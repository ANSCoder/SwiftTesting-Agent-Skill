# Example: ViewModel Tests

Full examples for testing `@MainActor`-isolated ViewModels using Swift Testing.

---

## The ViewModel under test

```swift
// UserListViewModel.swift
@MainActor
final class UserListViewModel: ObservableObject {
    @Published var users: [User] = []
    @Published var isLoading = false
    @Published var errorMessage: String?

    private let service: UserServiceProtocol
    private var loadTask: Task<Void, Never>?

    init(service: UserServiceProtocol) {
        self.service = service
    }

    func load() async {
        loadTask?.cancel()
        isLoading = true
        errorMessage = nil

        loadTask = Task {
            defer { isLoading = false }
            do {
                users = try await service.fetchAll()
            } catch {
                errorMessage = error.localizedDescription
            }
        }
        await loadTask?.value
    }

    func refresh() async {
        await load()
    }
}
```

---

## Mock service

```swift
// MockUserService.swift
final class MockUserService: UserServiceProtocol {
    var fetchAllResult: Result<[User], Error> = .success([])
    var fetchAllCallCount = 0

    func fetchAll() async throws -> [User] {
        fetchAllCallCount += 1
        return try fetchAllResult.get()
    }
}
```

---

## Full test suite

```swift
import Testing
@testable import MyApp

@Suite("UserListViewModel")
@MainActor
struct UserListViewModelTests {
    let mockService: MockUserService
    let sut: UserListViewModel

    init() {
        mockService = MockUserService()
        sut = UserListViewModel(service: mockService)
    }

    // MARK: - Initial state

    @Test func startsEmpty() {
        #expect(sut.users.isEmpty)
        #expect(sut.isLoading == false)
        #expect(sut.errorMessage == nil)
    }

    // MARK: - Successful load

    @Test func loadPopulatesUsers() async {
        mockService.fetchAllResult = .success([.fixture, .fixture2])
        await sut.load()
        #expect(sut.users.count == 2)
        #expect(sut.isLoading == false)
        #expect(sut.errorMessage == nil)
    }

    @Test func loadCallsServiceExactlyOnce() async {
        await sut.load()
        #expect(mockService.fetchAllCallCount == 1)
    }

    @Test func refreshCallsServiceAgain() async {
        await sut.load()
        await sut.refresh()
        #expect(mockService.fetchAllCallCount == 2)
    }

    // MARK: - Loading state

    @Test func setsIsLoadingDuringFetch() async {
        var loadingStates: [Bool] = []
        
        // Capture loading state changes via confirmation
        await confirmation("loading completes") { confirm in
            mockService.fetchAllResult = .success([.fixture])
            
            Task {
                await sut.load()
                confirm()
            }
            
            // Capture state mid-load
            await Task.yield()
            loadingStates.append(sut.isLoading)
        }

        loadingStates.append(sut.isLoading)
        #expect(loadingStates.contains(true))    // was loading at some point
        #expect(loadingStates.last == false)      // finished loading
    }

    // MARK: - Error handling

    @Test func setsErrorMessageOnFailure() async {
        mockService.fetchAllResult = .failure(NetworkError.timeout)
        await sut.load()
        #expect(sut.errorMessage != nil)
        #expect(sut.users.isEmpty)
        #expect(sut.isLoading == false)
    }

    @Test func clearsErrorMessageOnSuccessfulRetry() async {
        // First load fails
        mockService.fetchAllResult = .failure(NetworkError.timeout)
        await sut.load()
        #expect(sut.errorMessage != nil)

        // Retry succeeds
        mockService.fetchAllResult = .success([.fixture])
        await sut.load()
        #expect(sut.errorMessage == nil)
        #expect(!sut.users.isEmpty)
    }

    // MARK: - Parameterized error scenarios

    @Test(arguments: [
        NetworkError.timeout,
        NetworkError.unauthorized,
        NetworkError.badResponse,
    ])
    func setsErrorMessageForAllNetworkErrors(error: NetworkError) async {
        mockService.fetchAllResult = .failure(error)
        await sut.load()
        #expect(sut.errorMessage != nil)
    }
}
```

---

## Testing @Published property observation

When you need to verify that published properties change in a specific order:

```swift
@Test func publishedUsersUpdatesAfterLoad() async throws {
    var publishedValues: [[User]] = []
    mockService.fetchAllResult = .success([.fixture])

    let cancellable = sut.$users.sink { publishedValues.append($0) }
    defer { cancellable.cancel() }

    await sut.load()

    // Should have seen: initial empty state, then populated state
    #expect(publishedValues.count >= 2)
    #expect(publishedValues.first?.isEmpty == true)
    #expect(publishedValues.last?.count == 1)
}
```

---

## Testing navigation / routing

```swift
@Suite("UserListViewModel — navigation")
@MainActor
struct UserListViewModelNavigationTests {
    let sut: UserListViewModel
    
    init() {
        sut = UserListViewModel(service: MockUserService())
    }

    @Test func selectingUserSetsSelectedUser() async {
        let user = User.fixture
        sut.select(user)
        #expect(sut.selectedUser?.id == user.id)
    }

    @Test func deselectClearsSelectedUser() {
        sut.select(.fixture)
        sut.deselect()
        #expect(sut.selectedUser == nil)
    }
}
```
