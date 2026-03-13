# Example: SwiftUI Tests

Testing SwiftUI-specific behavior using Swift Testing.
SwiftUI views are tested indirectly through their ViewModels, environment, and observable state.

> SwiftUI views themselves are not directly unit-testable — the correct approach is to
> test the logic that drives them. These patterns work for both SwiftUI and UIKit projects.

---

## Testing @Observable ViewModels (Swift 5.9+)

```swift
import Testing
@testable import MyApp

// @Observable replaces ObservableObject in modern SwiftUI
@Observable
final class ProductListViewModel {
    var products: [Product] = []
    var searchQuery: String = ""
    var isLoading = false
    var selectedProduct: Product?

    private let service: ProductServiceProtocol

    init(service: ProductServiceProtocol) {
        self.service = service
    }

    var filteredProducts: [Product] {
        guard !searchQuery.isEmpty else { return products }
        return products.filter { $0.name.localizedCaseInsensitiveContains(searchQuery) }
    }

    func load() async {
        isLoading = true
        defer { isLoading = false }
        products = (try? await service.fetchAll()) ?? []
    }
}

@Suite("ProductListViewModel")
@MainActor
struct ProductListViewModelTests {
    let mockService: MockProductService
    let sut: ProductListViewModel

    init() {
        mockService = MockProductService()
        sut = ProductListViewModel(service: mockService)
    }

    @Test func startsWithEmptyState() {
        #expect(sut.products.isEmpty, "Products should be empty before first load")
        #expect(sut.searchQuery.isEmpty, "Search query should start empty")
        #expect(!sut.isLoading, "Should not be loading before load() is called")
    }

    @Test func loadPopulatesProducts() async {
        mockService.stubbedProducts = [.fixture, .fixture2, .fixture3]
        await sut.load()
        #expect(sut.products.count == 3, "Should load all products from service")
    }

    @Test func filteredProductsReflectsSearchQuery() async {
        mockService.stubbedProducts = [
            Product(id: "1", name: "Swift Programming"),
            Product(id: "2", name: "Kotlin Basics"),
            Product(id: "3", name: "SwiftUI Mastery"),
        ]
        await sut.load()

        sut.searchQuery = "Swift"

        #expect(sut.filteredProducts.count == 2, "Filter should match 'Swift Programming' and 'SwiftUI Mastery'")
        #expect(sut.filteredProducts.allSatisfy { $0.name.localizedCaseInsensitiveContains("Swift") })
    }

    @Test func emptySearchQueryShowsAllProducts() async {
        mockService.stubbedProducts = [.fixture, .fixture2]
        await sut.load()

        sut.searchQuery = "xyz"
        sut.searchQuery = ""

        #expect(
            sut.filteredProducts.count == sut.products.count,
            "Clearing search query should show all products"
        )
    }

    @Test(arguments: ["", " ", "swift", "SWIFT", "SwIfT"])
    func searchIsCaseInsensitive(query: String) async {
        mockService.stubbedProducts = [Product(id: "1", name: "Swift Guide")]
        await sut.load()
        sut.searchQuery = query

        if query.trimmingCharacters(in: .whitespaces).isEmpty {
            #expect(sut.filteredProducts.count == 1, "Empty/whitespace query shows all products")
        } else {
            #expect(!sut.filteredProducts.isEmpty, "Case-insensitive search for '\(query)' should find 'Swift Guide'")
        }
    }
}
```

---

## Testing @Environment and dependency injection

```swift
// AppEnvironment — SwiftUI environment object
struct AppEnvironment {
    var router: AppRouter
    var featureFlags: FeatureFlags
    var analytics: AnalyticsProtocol
}

@Suite("Feature flag gating")
struct FeatureFlagTests {

    @Test func premiumFeatureHiddenWithoutFlag() {
        var flags = FeatureFlags()
        flags.isPremiumEnabled = false
        let env = AppEnvironment(router: .mock, featureFlags: flags, analytics: .mock)
        let viewModel = DashboardViewModel(environment: env)

        #expect(!viewModel.showsPremiumSection, "Premium section should be hidden when flag is disabled")
    }

    @Test func premiumFeatureVisibleWithFlag() {
        var flags = FeatureFlags()
        flags.isPremiumEnabled = true
        let env = AppEnvironment(router: .mock, featureFlags: flags, analytics: .mock)
        let viewModel = DashboardViewModel(environment: env)

        #expect(viewModel.showsPremiumSection, "Premium section should appear when feature flag is enabled")
    }
}
```

---

## Testing navigation / routing (SwiftUI NavigationPath)

```swift
@Observable
final class AppRouter {
    var path: NavigationPath = NavigationPath()

    func navigate(to destination: AppDestination) {
        path.append(destination)
    }

    func pop() {
        guard !path.isEmpty else { return }
        path.removeLast()
    }

    func popToRoot() {
        path = NavigationPath()
    }
}

@Suite("AppRouter")
@MainActor
struct AppRouterTests {
    let sut = AppRouter()

    @Test func navigatingAppendsToPath() {
        sut.navigate(to: .productDetail(id: "abc"))
        #expect(sut.path.count == 1, "Navigation should add one item to the path")
    }

    @Test func popRemovesLastDestination() {
        sut.navigate(to: .productDetail(id: "abc"))
        sut.navigate(to: .checkout)
        sut.pop()
        #expect(sut.path.count == 1, "Pop should remove the last destination only")
    }

    @Test func popToRootClearsEntirePath() {
        sut.navigate(to: .productDetail(id: "abc"))
        sut.navigate(to: .checkout)
        sut.navigate(to: .orderConfirmation)
        sut.popToRoot()
        #expect(sut.path.isEmpty, "popToRoot should clear all destinations")
    }

    @Test func poppingEmptyPathDoesNotCrash() {
        #expect(sut.path.isEmpty)
        sut.pop()  // should not throw or crash
        #expect(sut.path.isEmpty, "Popping an empty path should be a no-op")
    }
}
```

---

## Testing form validation (common SwiftUI pattern)

```swift
@Observable
final class RegistrationViewModel {
    var email: String = ""
    var password: String = ""
    var confirmPassword: String = ""

    var isFormValid: Bool {
        isEmailValid && isPasswordValid && passwordsMatch
    }

    var isEmailValid: Bool {
        email.contains("@") && email.contains(".")
    }

    var isPasswordValid: Bool {
        password.count >= 8
    }

    var passwordsMatch: Bool {
        password == confirmPassword && !password.isEmpty
    }

    var validationSummary: [String] {
        var errors: [String] = []
        if !isEmailValid { errors.append("Enter a valid email address") }
        if !isPasswordValid { errors.append("Password must be at least 8 characters") }
        if !passwordsMatch { errors.append("Passwords do not match") }
        return errors
    }
}

@Suite("Registration form validation")
struct RegistrationViewModelTests {
    let sut = RegistrationViewModel()

    @Test func formIsInvalidWhenEmpty() {
        #expect(!sut.isFormValid, "Empty form should not be valid")
        #expect(sut.validationSummary.count == 3, "Empty form should have 3 validation errors")
    }

    @Test(arguments: [
        ("notanemail", false),
        ("missing@dot", false),
        ("valid@example.com", true),
        ("user+tag@domain.co.uk", true),
    ])
    func emailValidation(email: String, isValid: Bool) {
        sut.email = email
        #expect(
            sut.isEmailValid == isValid,
            "Email '\(email)' should be \(isValid ? "valid" : "invalid")"
        )
    }

    @Test(arguments: [
        ("short", false),
        ("exactly8", true),
        ("longpassword123", true),
    ])
    func passwordLengthValidation(password: String, isValid: Bool) {
        sut.password = password
        #expect(
            sut.isPasswordValid == isValid,
            "Password of length \(password.count) should be \(isValid ? "valid" : "invalid")"
        )
    }

    @Test func formIsValidWithCorrectInput() {
        sut.email = "user@example.com"
        sut.password = "securePassword1"
        sut.confirmPassword = "securePassword1"
        #expect(sut.isFormValid, "Form should be valid when all fields are correctly filled")
        #expect(sut.validationSummary.isEmpty, "Valid form should have no validation errors")
    }
}
```

---

## Testing async image loading (common SwiftUI pattern)

```swift
@Observable
final class AsyncImageViewModel {
    var imageState: ImageState = .idle
    private let loader: ImageLoaderProtocol

    enum ImageState: Equatable {
        case idle, loading, loaded(Data), failed(String)
    }

    init(loader: ImageLoaderProtocol) {
        self.loader = loader
    }

    func load(url: URL) async {
        imageState = .loading
        do {
            let data = try await loader.load(from: url)
            imageState = .loaded(data)
        } catch {
            imageState = .failed(error.localizedDescription)
        }
    }
}

@Suite("AsyncImageViewModel")
@MainActor
struct AsyncImageViewModelTests {
    let mockLoader: MockImageLoader
    let sut: AsyncImageViewModel

    init() {
        mockLoader = MockImageLoader()
        sut = AsyncImageViewModel(loader: mockLoader)
    }

    @Test func transitionsToLoadingThenLoaded() async throws {
        let imageData = Data(repeating: 0xFF, count: 1024)
        mockLoader.stubbedResult = .success(imageData)

        await sut.load(url: URL.fixture)

        if case .loaded(let data) = sut.imageState {
            #expect(data == imageData, "Loaded image data should match what the loader returned")
        } else {
            Issue.record("Expected .loaded state but got \(sut.imageState)")
        }
    }

    @Test func transitionsToFailedOnError() async {
        mockLoader.stubbedResult = .failure(URLError(.notConnectedToInternet))
        await sut.load(url: URL.fixture)

        if case .failed(let message) = sut.imageState {
            #expect(!message.isEmpty, "Failure state should include an error description")
        } else {
            Issue.record("Expected .failed state but got \(sut.imageState)")
        }
    }
}
```

---

## SwiftUI vs UIKit — what changes in tests

| Concern | SwiftUI | UIKit |
|---|---|---|
| ViewModel type | `@Observable` or `ObservableObject` | Same |
| Setup | `init()` in `@Suite` | Same |
| Main thread | `@MainActor` on suite or test | Same |
| Navigation | Test `NavigationPath` / router | Test `UINavigationController` push/pop |
| Delegate testing | Via coordinator in `UIViewRepresentable` | Direct on `UIViewController` |
| Form validation | Computed properties on ViewModel | Same pattern |
| View rendering | Not unit-testable — test the model | ViewControllers can be tested with `loadViewIfNeeded()` |
