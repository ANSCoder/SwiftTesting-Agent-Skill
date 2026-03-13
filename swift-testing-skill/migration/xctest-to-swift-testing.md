# Migration: XCTest → Swift Testing

A complete side-by-side reference for converting existing test files.
Use this when refactoring legacy test code or reviewing AI-generated output.

---

## File header

```swift
// BEFORE
import XCTest
@testable import MyApp

// AFTER
import Testing
@testable import MyApp
```

---

## Test class / suite

```swift
// BEFORE
class UserServiceTests: XCTestCase { }

// AFTER
@Suite("User service")
struct UserServiceTests { }
```

---

## Setup and teardown

```swift
// BEFORE
class OrderTests: XCTestCase {
    var sut: OrderService!
    var mockDB: MockDatabase!

    override func setUpWithError() throws {
        try super.setUpWithError()
        mockDB = MockDatabase()
        sut = OrderService(database: mockDB)
    }

    override func tearDownWithError() throws {
        sut = nil
        mockDB = nil
        try super.tearDownWithError()
    }
}

// AFTER
@Suite("Order service")
struct OrderTests {
    let sut: OrderService
    let mockDB: MockDatabase

    init() {
        mockDB = MockDatabase()
        sut = OrderService(database: mockDB)
    }
    // deinit is automatic for structs — no manual teardown needed
    // for class-backed resources, implement deinit explicitly
}
```

---

## Test function names

```swift
// BEFORE
func testFetchOrderReturnsCorrectTotal() { }
func testFetchOrderThrowsWhenNotFound() { }

// AFTER
@Test func fetchOrderReturnsCorrectTotal() { }
@Test func fetchOrderThrowsWhenNotFound() { }
```

---

## Assertions — equality and comparison

```swift
// BEFORE
XCTAssertEqual(order.total, 99.99)
XCTAssertNotEqual(order.id, "")
XCTAssertGreaterThan(items.count, 0)
XCTAssertLessThanOrEqual(discountRate, 0.5)

// AFTER
#expect(order.total == 99.99)
#expect(order.id != "")
#expect(items.count > 0)
#expect(discountRate <= 0.5)
```

---

## Assertions — boolean

```swift
// BEFORE
XCTAssertTrue(order.isConfirmed)
XCTAssertFalse(order.isCancelled)

// AFTER
#expect(order.isConfirmed)
#expect(!order.isCancelled)
```

---

## Assertions — nil checks

```swift
// BEFORE
XCTAssertNil(order.errorMessage)
XCTAssertNotNil(order.confirmationNumber)

// AFTER
#expect(order.errorMessage == nil)
#expect(order.confirmationNumber != nil)
```

---

## Unwrapping optionals

```swift
// BEFORE
let number = try XCTUnwrap(order.confirmationNumber)
XCTAssertEqual(number.count, 8)

// AFTER
let number = try #require(order.confirmationNumber)
#expect(number.count == 8)
```

---

## Throwing assertions

```swift
// BEFORE
XCTAssertThrowsError(try service.placeOrder(.empty)) { error in
    XCTAssertEqual(error as? OrderError, .emptyCart)
}
XCTAssertNoThrow(try service.placeOrder(.valid))

// AFTER
#expect(throws: OrderError.emptyCart) {
    try service.placeOrder(.empty)
}
// "no throw" — just call it. Test fails naturally if it throws.
try service.placeOrder(.valid)
```

---

## Async tests

```swift
// BEFORE
func testFetchOrderAsync() {
    let exp = expectation(description: "fetch completes")
    service.fetchOrder(id: "abc") { result in
        switch result {
        case .success(let order):
            XCTAssertEqual(order.id, "abc")
        case .failure:
            XCTFail("Unexpected failure")
        }
        exp.fulfill()
    }
    waitForExpectations(timeout: 5)
}

// AFTER
@Test func fetchesOrderByID() async throws {
    let order = try await service.fetchOrder(id: "abc")
    #expect(order.id == "abc")
}
```

---

## Notification / callback-based async

```swift
// BEFORE
func testOrderConfirmationNotificationPosted() {
    let exp = expectation(forNotification: .orderConfirmed, object: nil)
    service.confirmOrder(id: "abc") { _ in }
    waitForExpectations(timeout: 5)
}

// AFTER
@Test func orderConfirmationNotificationPosted() async throws {
    await confirmation("order confirmed notification") { confirm in
        NotificationCenter.default.addObserver(
            forName: .orderConfirmed, object: nil, queue: nil
        ) { _ in confirm() }
        try await service.confirmOrder(id: "abc")
    }
}
```

---

## Repeated test methods → parameterized

```swift
// BEFORE
func testValidatesVisaCard() { XCTAssertTrue(validator.isValid(.visa)) }
func testValidatesMastercard() { XCTAssertTrue(validator.isValid(.mastercard)) }
func testValidatesAmex() { XCTAssertTrue(validator.isValid(.amex)) }

// AFTER
@Test(arguments: CardType.allCases)
func validatesAllSupportedCardTypes(cardType: CardType) {
    #expect(validator.isValid(cardType))
}
```

---

## Manually failing a test

```swift
// BEFORE
XCTFail("This code path should never be reached")

// AFTER
Issue.record("This code path should never be reached")
```

---

## Skipping / disabling tests

```swift
// BEFORE
try XCTSkipIf(ProcessInfo.processInfo.environment["CI"] != nil)

// AFTER
@Test(.disabled("Skipped in CI — requires live network"))
func liveNetworkIntegrationTest() async throws { ... }
```

---

## Full file: before and after

```swift
// ====== BEFORE ======
import XCTest
@testable import MyApp

class CartServiceTests: XCTestCase {
    var sut: CartService!

    override func setUp() {
        sut = CartService()
    }

    func testAddItemIncreasesCount() {
        sut.add(.fixture)
        XCTAssertEqual(sut.items.count, 1)
    }

    func testRemoveItemDecreasesCount() {
        sut.add(.fixture)
        sut.remove(at: 0)
        XCTAssertTrue(sut.items.isEmpty)
    }

    func testCheckoutAsync() {
        let exp = expectation(description: "checkout")
        sut.add(.fixture)
        sut.checkout { result in
            XCTAssertNotNil(try? result.get())
            exp.fulfill()
        }
        waitForExpectations(timeout: 5)
    }
}

// ====== AFTER ======
import Testing
@testable import MyApp

@Suite("Cart service")
struct CartServiceTests {
    let sut: CartService

    init() {
        sut = CartService()
    }

    @Test func addingItemIncreasesCount() {
        sut.add(.fixture)
        #expect(sut.items.count == 1)
    }

    @Test func removingItemDecreasesCount() {
        sut.add(.fixture)
        sut.remove(at: 0)
        #expect(sut.items.isEmpty)
    }

    @Test func checkoutCompletesSuccessfully() async throws {
        sut.add(.fixture)
        let receipt = try await sut.checkout()
        #expect(receipt != nil)
    }
}
```
