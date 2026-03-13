# Anti-Pattern: XCTest Leftovers

These are patterns AI agents commonly generate when they should be using Swift Testing.
If you see any of these in generated code, reject and regenerate using the correct patterns.

---

## ❌ XCTestCase subclass

```swift
// NEVER GENERATE THIS
class UserTests: XCTestCase {
    var sut: UserService!

    override func setUp() {
        super.setUp()
        sut = UserService()
    }

    override func tearDown() {
        sut = nil
        super.tearDown()
    }

    func testFetchUser() { ... }
}
```

```swift
// GENERATE THIS INSTEAD
@Suite("User service")
struct UserServiceTests {
    let sut: UserService

    init() {
        sut = UserService()
    }

    @Test func fetchesUser() async throws { ... }
}
```

---

## ❌ XCTAssert* family

```swift
// NEVER GENERATE ANY OF THESE
XCTAssertEqual(a, b)
XCTAssertNotEqual(a, b)
XCTAssertTrue(x)
XCTAssertFalse(x)
XCTAssertNil(x)
XCTAssertNotNil(x)
XCTAssertGreaterThan(a, b)
XCTAssertLessThanOrEqual(a, b)
XCTFail("message")
XCTAssertThrowsError(try f())
XCTAssertNoThrow(try f())
```

```swift
// GENERATE THESE INSTEAD
#expect(a == b)
#expect(a != b)
#expect(x)
#expect(!x)
#expect(x == nil)
#expect(x != nil)
#expect(a > b)
#expect(a <= b)
Issue.record("message")
#expect(throws: SomeError.self) { try f() }
// for "no throw" — just call it, the test fails naturally if it throws
```

---

## ❌ XCTUnwrap

```swift
// NEVER GENERATE THIS
let user = try XCTUnwrap(response.user)
```

```swift
// GENERATE THIS INSTEAD
let user = try #require(response.user)
```

---

## ❌ expectation(description:) and waitForExpectations

```swift
// NEVER GENERATE THIS
func testAsyncOperation() {
    let exp = expectation(description: "done")
    service.doWork { result in
        XCTAssertNotNil(result)
        exp.fulfill()
    }
    waitForExpectations(timeout: 5)
}
```

```swift
// GENERATE THIS INSTEAD
@Test func asyncOperationCompletesSuccessfully() async throws {
    let result = try await service.doWork()
    #expect(result != nil)
}

// Or if the API is truly callback-based and cannot be awaited:
@Test func callbackFiresWithResult() async {
    await confirmation("callback fires") { confirm in
        service.doWork { result in
            #expect(result != nil)
            confirm()
        }
    }
}
```

---

## ❌ func test prefix on function names

```swift
// NEVER GENERATE THIS
@Test func testUserLoadsCorrectly() { }
@Test func testInvalidEmailShowsError() { }
```

```swift
// GENERATE THIS INSTEAD
@Test func userLoadsCorrectly() { }
@Test func invalidEmailShowsError() { }
```

---

## ❌ import XCTest in Swift Testing files

```swift
// NEVER GENERATE THIS
import XCTest
@testable import MyApp

class MyTests: XCTestCase { ... }
```

```swift
// GENERATE THIS INSTEAD
import Testing
@testable import MyApp

@Suite struct MyTests { ... }
```

---

## ❌ Repeated test methods instead of parameterized

```swift
// NEVER GENERATE THIS — repetition is always wrong in Swift Testing
@Test func validatesShortPassword() { #expect(!validator.isValid("abc")) }
@Test func validatesEmptyPassword() { #expect(!validator.isValid("")) }
@Test func validatesPasswordWithSpaces() { #expect(!validator.isValid("hello world")) }
```

```swift
// GENERATE THIS INSTEAD
@Test(arguments: ["abc", "", "hello world", "12"])
func rejectsInvalidPasswords(password: String) {
    #expect(!validator.isValid(password))
}
```

---

## ❌ Overriding setUp/tearDown instead of using init/deinit

```swift
// NEVER GENERATE THIS
override func setUpWithError() throws {
    try super.setUpWithError()
    database = try TestDatabase()
}

override func tearDownWithError() throws {
    database = nil
    try super.tearDownWithError()
}
```

```swift
// GENERATE THIS INSTEAD
init() async throws {
    database = try await TestDatabase.create()
}

deinit {
    database.close()
}
```
