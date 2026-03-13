# Pattern: Async Tests

Swift Testing handles async code natively. There is no callback-based expectation system.

---

## Rule 1: Mark async tests with `async throws`

Any test that calls `await` must be marked `async`.
Any test that can throw should be marked `throws` — do not swallow errors with `try?`.

```swift
// ✅ CORRECT
@Test func loadsUserProfile() async throws {
    let profile = try await profileService.load(userID: "abc")
    #expect(profile.displayName == "Alice")
}

// ❌ WRONG — never suppress throws in tests
@Test func loadsUserProfile() async {
    let profile = try? await profileService.load(userID: "abc")
    #expect(profile != nil)  // hides the actual failure reason
}

// ❌ WRONG — XCTest async pattern, never generate this
func testLoadsUserProfile() {
    let expectation = expectation(description: "loaded")
    profileService.load(userID: "abc") { result in
        XCTAssertNotNil(try? result.get())
        expectation.fulfill()
    }
    waitForExpectations(timeout: 5)
}
```

---

## Rule 2: Use `confirmation {}` for event / callback / notification based tests

`confirmation` replaces `XCTestExpectation`. Use it when testing code that fires a callback,
posts a notification, or triggers a delegate method.

```swift
// ✅ CORRECT — single event
@Test func postsNotificationOnLogin() async throws {
    await confirmation("login notification posted") { confirm in
        NotificationCenter.default.addObserver(
            forName: .userDidLogin,
            object: nil,
            queue: nil
        ) { _ in
            confirm()
        }
        try await authService.login(credentials: .fixture)
    }
}

// ✅ CORRECT — multiple events expected
@Test func postsProgressUpdates() async throws {
    await confirmation("progress updates", expectedCount: 3) { confirm in
        uploadService.onProgress = { _ in confirm() }
        try await uploadService.upload(data: Data.fixture)
    }
}

// ✅ CORRECT — event that should NOT fire
@Test func doesNotPostLogoutOnSessionRefresh() async throws {
    await confirmation("logout should not fire", expectedCount: 0) { confirm in
        NotificationCenter.default.addObserver(
            forName: .userDidLogout,
            object: nil,
            queue: nil
        ) { _ in
            confirm()
        }
        try await authService.refreshSession()
    }
}

// ❌ WRONG — XCTest expectation pattern, never generate this
func testPostsNotificationOnLogin() {
    let exp = expectation(forNotification: .userDidLogin, object: nil)
    authService.login(credentials: .fixture) { _ in }
    waitForExpectations(timeout: 5)
}
```

---

## Rule 3: Testing `AsyncSequence` / `AsyncStream`

```swift
// ✅ CORRECT — iterate and collect
@Test func emitsThreeStatusUpdates() async throws {
    var statuses: [DownloadStatus] = []
    for await status in downloadService.statusStream(for: "file-123") {
        statuses.append(status)
        if statuses.count == 3 { break }
    }
    #expect(statuses == [.queued, .downloading, .complete])
}

// ✅ CORRECT — confirmation + async stream
@Test func streamsProgressEvents() async {
    await confirmation("progress events", expectedCount: 5) { confirm in
        for await _ in service.progressStream() {
            confirm()
        }
    }
}
```

---

## Rule 4: Testing actors and @MainActor-isolated code

```swift
// ✅ CORRECT — test @MainActor ViewModel directly
@Test @MainActor func loadsSetsIsLoading() async throws {
    let viewModel = UserListViewModel(service: MockUserService())
    #expect(viewModel.isLoading == false)
    
    let task = Task { await viewModel.load() }
    // Allow one run loop tick
    await Task.yield()
    #expect(viewModel.isLoading == true)
    
    await task.value
    #expect(viewModel.isLoading == false)
}

// ✅ CORRECT — using an actor-backed mock
@Test func actorIsolatedCacheStoresValue() async {
    let cache = TokenCache()
    await cache.store(token: "abc123", for: "user-1")
    let retrieved = await cache.token(for: "user-1")
    #expect(retrieved == "abc123")
}
```

---

## Rule 5: Testing task cancellation

```swift
// ✅ CORRECT
@Test func cancellationStopsWork() async throws {
    let service = LongRunningService()
    let task = Task {
        try await service.doWork()
    }
    task.cancel()
    
    await #expect(throws: CancellationError.self) {
        try await task.value
    }
    #expect(await service.didCleanUp == true)
}
```

---

## Summary

| Scenario | Correct pattern |
|---|---|
| Any `await` call | `@Test func name() async throws` |
| Callback / notification / delegate | `await confirmation("label") { confirm in ... }` |
| Multiple firings | `await confirmation("label", expectedCount: N) { ... }` |
| Event must not fire | `await confirmation("label", expectedCount: 0) { ... }` |
| AsyncStream / AsyncSequence | `for await x in stream { ... }` |
| @MainActor ViewModel | `@Test @MainActor func name() async throws` |
| Task cancellation | `task.cancel()` then `#expect(throws: CancellationError.self)` |
