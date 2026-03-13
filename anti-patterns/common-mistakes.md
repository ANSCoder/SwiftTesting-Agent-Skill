# Anti-Pattern: Common Swift Testing Mistakes

These mistakes appear even in code that has already dropped XCTest.
They look like Swift Testing but are still wrong.

---

## ❌ Swallowing errors with try? in async tests

```swift
// WRONG — hides the failure reason completely
@Test func loadsData() async {
    let result = try? await service.load()
    #expect(result != nil)
}

// CORRECT — let the error propagate naturally
@Test func loadsData() async throws {
    let result = try await service.load()
    #expect(result != nil)
}
```

---

## ❌ Using confirmation when await is available

`confirmation` is for callback-based APIs that cannot be awaited.
If the API has an async version, always use that instead.

```swift
// WRONG — unnecessary confirmation when async API exists
@Test func fetchesUser() async throws {
    await confirmation("fetched") { confirm in
        Task {
            _ = try await userService.fetch(id: "1")
            confirm()
        }
    }
}

// CORRECT
@Test func fetchesUser() async throws {
    let user = try await userService.fetch(id: "1")
    #expect(user.id == "1")
}
```

---

## ❌ Force unwrapping instead of #require

```swift
// WRONG — crash instead of a useful test failure
@Test func parsesResponse() throws {
    let parsed = ResponseParser.parse(data: .fixture)
    let first = parsed!.items.first!
    #expect(first.id == "abc")
}

// CORRECT
@Test func parsesResponse() throws {
    let parsed = try #require(ResponseParser.parse(data: .fixture))
    let first = try #require(parsed.items.first)
    #expect(first.id == "abc")
}
```

---

## ❌ Checking `throws` manually instead of using #expect(throws:)

```swift
// WRONG — manual do/catch is verbose and error-prone
@Test func throwsOnInvalidInput() async throws {
    do {
        try await service.process(input: .invalid)
        Issue.record("Expected error was not thrown")
    } catch let error as ProcessingError {
        #expect(error == .invalidInput)
    }
}

// CORRECT
@Test func throwsOnInvalidInput() async {
    await #expect(throws: ProcessingError.invalidInput) {
        try await service.process(input: .invalid)
    }
}
```

---

## ❌ Shared mutable state across tests

Swift Testing can run tests in parallel by default.
Shared global state causes flaky tests.

```swift
// WRONG — static var shared across parallel tests
@Suite
struct CacheTests {
    static var cache = Cache()  // shared — will cause race conditions

    @Test func storesValue() { CacheTests.cache.set("a", value: 1) }
    @Test func removesValue() { CacheTests.cache.remove("a") }
}

// CORRECT — each test gets its own instance via init()
@Suite
struct CacheTests {
    let cache: Cache

    init() { cache = Cache() }

    @Test func storesValue() { cache.set("a", value: 1) }
    @Test func removesValue() { cache.remove("a") }
}
```

---

## ❌ Using @Suite on a class

```swift
// WRONG — @Suite on a class doesn't get per-test init isolation
@Suite
class OrderTests {
    var order = Order()
    @Test func addsItem() { order.add(.fixture) }
}

// CORRECT — use a struct
@Suite
struct OrderTests {
    let order: Order
    init() { order = Order() }
    @Test func addsItem() { order.add(.fixture) }
}
```

---

## ❌ Forgetting @MainActor on ViewModel tests

If a ViewModel is `@MainActor`-isolated, tests that access it must also be on `@MainActor`.

```swift
// WRONG — accessing @MainActor ViewModel off main thread
@Test func viewModelLoadsUsers() async throws {
    let vm = UserListViewModel()  // @MainActor — can only be created on main
    await vm.load()
    #expect(!vm.users.isEmpty)
}

// CORRECT
@Test @MainActor func viewModelLoadsUsers() async throws {
    let vm = UserListViewModel()
    await vm.load()
    #expect(!vm.users.isEmpty)
}

// ALSO CORRECT — annotate the whole suite
@Suite @MainActor
struct UserListViewModelTests {
    let vm = UserListViewModel()

    @Test func loadsUsers() async throws {
        await vm.load()
        #expect(!vm.users.isEmpty)
    }
}
```

---

## ❌ Using sleep for timing instead of structured concurrency

```swift
// WRONG — brittle, slow, flaky
@Test func debounceDelaysUpdate() async throws {
    viewModel.search(query: "swift")
    try await Task.sleep(nanoseconds: 500_000_000)  // hope for the best
    #expect(viewModel.results.count > 0)
}

// CORRECT — test the observable outcome, not timing
@Test func debounceEventuallyFires() async throws {
    await confirmation("search completes") { confirm in
        viewModel.onResultsUpdated = { _ in confirm() }
        viewModel.search(query: "swift")
    }
}
```
