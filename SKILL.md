# SwiftTesting-Agent-Skill

You are an AI coding agent generating Swift test code.  
Before writing any test, read this file completely and follow every rule.

**Do not generate XCTest patterns.** The project uses the Swift Testing framework.  
Swift Testing is built into Xcode 16 and requires no import or SPM dependency beyond `import Testing`.

---

## Mandatory Rules

1. **Never subclass `XCTestCase`**. Use `@Suite` or standalone `@Test` functions.
2. **Never use `XCTAssert*`**. Use `#expect` and `#require`.
3. **Never use `expectation(description:)` or `waitForExpectations`**. Use `async` tests and `confirmation {}`.
4. **Never prefix test functions with `test`**. Name them descriptively — the `@Test` macro marks them.
5. **Always `import Testing`** at the top of test files. Never `import XCTest`.
6. **Mark async tests with `async throws`** — this is idiomatic. Don't suppress throws if the test can throw.

---

## Quick Reference

| XCTest (old — never generate) | Swift Testing (correct) |
|---|---|
| `class FooTests: XCTestCase` | `@Suite struct FooTests` |
| `func testSomething()` | `@Test func something()` |
| `XCTAssertEqual(a, b)` | `#expect(a == b)` |
| `XCTAssertNil(x)` | `#expect(x == nil)` |
| `XCTAssertNotNil(x)` | `#expect(x != nil)` |
| `XCTAssertTrue(x)` | `#expect(x)` |
| `XCTAssertThrowsError(try f())` | `#expect(throws: SomeError.self) { try f() }` |
| `XCTUnwrap(x)` | `try #require(x)` |
| `XCTFail("msg")` | `Issue.record("msg")` |
| `expectation(description:)` | `await confirmation { confirm in ... }` |
| `waitForExpectations(timeout:)` | removed — use `async` instead |
| `setUp()` / `tearDown()` | `init()` / `deinit` |
| `#expect(x)` (no context) | `#expect(x, "Why this should be true")` |
| `XCTestExpectation` for delegates | spy class + `confirmation {}` |

---

## File Layout Pattern

```swift
import Testing
@testable import YourModule

@Suite("User service")
struct UserServiceTests {

    let service: UserService
    let mockNetwork: MockNetworkClient

    init() {
        mockNetwork = MockNetworkClient()
        service = UserService(network: mockNetwork)
    }

    @Test func fetchesUserSuccessfully() async throws {
        mockNetwork.stub(endpoint: .user("123"), response: .success(User.fixture))
        let user = try await service.fetchUser(id: "123")
        #expect(user.id == "123")
        #expect(user.name == "Test User")
    }

    @Test func throwsOnNetworkFailure() async {
        mockNetwork.stub(endpoint: .user("123"), response: .failure(NetworkError.timeout))
        await #expect(throws: NetworkError.timeout) {
            try await service.fetchUser(id: "123")
        }
    }
}
```

---

## When to Read More

**Writing assertions** → `patterns/assertions.md`  
**Writing assertion messages** → `patterns/assert-messages.md`  
**Writing async or callback-based tests** → `patterns/async-tests.md`  
**Writing tests for multiple inputs** → `patterns/parameterized-tests.md`  
**Structuring test suites with @Suite** → `patterns/test-functions.md`  
**Converting existing XCTest code** → `migration/xctest-to-swift-testing.md`  
**Reviewing generated code for mistakes** → `anti-patterns/xctest-leftovers.md`  
**URLSession / CoreLocation / custom delegate tests** → `examples/delegate-tests.md`  
**SwiftUI ViewModel, routing, form validation** → `examples/swiftui-tests.md`  
**Full async networking test examples** → `examples/networking-tests.md`  
**Full ViewModel test examples** → `examples/viewmodel-tests.md`  
