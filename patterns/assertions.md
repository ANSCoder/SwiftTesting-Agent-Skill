# Pattern: Assertions

Swift Testing replaces the entire `XCTAssert*` family with two macros: `#expect` and `#require`.

---

## #expect — soft assertion (test continues on failure)

`#expect` evaluates a condition and records a failure if false, but the test continues running.
Use it for most assertions.

```swift
// ✅ CORRECT
#expect(user.name == "Alice")
#expect(items.count == 3)
#expect(result != nil)
#expect(isValid)
#expect(!errors.isEmpty)

// ❌ WRONG — XCTest patterns, never generate these
XCTAssertEqual(user.name, "Alice")
XCTAssertNotNil(result)
XCTAssertTrue(isValid)
```

---

## #require — hard assertion (test stops on failure)

`#require` throws if the condition fails, stopping the test immediately.
Use it when subsequent lines depend on this value being non-nil or valid.

```swift
// ✅ CORRECT — unwraps optional, stops if nil
let user = try #require(response.user)
#expect(user.name == "Alice")  // only runs if user is non-nil

// ✅ CORRECT — also used as a condition gate
try #require(!items.isEmpty)
let first = items[0]  // safe to access

// ❌ WRONG — XCTest equivalent
let user = try XCTUnwrap(response.user)
```

---

## Throwing assertions

```swift
// ✅ CORRECT — assert a specific error type is thrown
#expect(throws: NetworkError.self) {
    try service.fetchWithoutAuth()
}

// ✅ CORRECT — assert a specific error value is thrown
#expect(throws: NetworkError.unauthorized) {
    try service.fetchWithoutAuth()
}

// ✅ CORRECT — inspect the thrown error
#expect {
    try service.fetchWithoutAuth()
} throws: { error in
    guard let networkError = error as? NetworkError else { return false }
    return networkError == .unauthorized
}

// ✅ CORRECT — async throwing
await #expect(throws: NetworkError.timeout) {
    try await service.fetch(endpoint: .users)
}

// ❌ WRONG — XCTest patterns
XCTAssertThrowsError(try service.fetchWithoutAuth())
XCTAssertNoThrow(try service.fetch())
```

---

## Collection assertions

```swift
// ✅ CORRECT
#expect(items.count == 3)
#expect(items.contains { $0.id == "abc" })
#expect(items.isEmpty)
#expect(!items.isEmpty)

// ✅ CORRECT — check first element without XCTUnwrap
let first = try #require(items.first)
#expect(first.name == "Alice")
```

---

## Floating point comparisons

```swift
// ✅ CORRECT — standard Swift comparison with tolerance
let result = calculator.divide(10, by: 3)
#expect(abs(result - 3.333) < 0.001)

// ✅ ALSO CORRECT — using isApproximatelyEqual if available
#expect(result.isApproximatelyEqual(to: 3.333, absoluteTolerance: 0.001))
```

---

## Negation

```swift
// ✅ CORRECT — negate inside #expect, not outside
#expect(user.isAdmin == false)
#expect(!flags.contains(.deprecated))

// ❌ WRONG — no XCTAssertFalse
XCTAssertFalse(user.isAdmin)
```

---

## Summary

| Use case | Macro |
|---|---|
| Most assertions | `#expect(condition)` |
| Unwrap optional, stop if nil | `try #require(optional)` |
| Assert throws specific type | `#expect(throws: ErrorType.self) { }` |
| Assert throws specific value | `#expect(throws: specificError) { }` |
| Inspect thrown error | `#expect { } throws: { error in ... }` |
| Stop test on condition failure | `try #require(condition)` |
