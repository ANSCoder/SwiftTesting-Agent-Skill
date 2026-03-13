# Pattern: Test Functions and Suites

How to structure test files, name tests, and use @Suite correctly.

---

## Rule 1: Use @Suite structs instead of XCTestCase classes

```swift
// ✅ CORRECT — value type, no inheritance, no lifecycle boilerplate
import Testing
@testable import MyApp

@Suite("Shopping cart")
struct ShoppingCartTests {
    let cart: ShoppingCart

    init() {
        cart = ShoppingCart()
    }

    @Test func startsEmpty() {
        #expect(cart.items.isEmpty)
        #expect(cart.total == 0)
    }
}

// ❌ WRONG — class + XCTestCase, never generate this
class ShoppingCartTests: XCTestCase {
    var cart: ShoppingCart!
    override func setUp() { cart = ShoppingCart() }
}
```

---

## Rule 2: Name tests descriptively — no `test` prefix

The `@Test` macro marks the function. The name should read like a sentence.

```swift
// ✅ CORRECT — reads like a sentence
@Test func addsItemToCart()
@Test func removingLastItemEmptiesCart()
@Test func discountCodeReducesTotalByPercentage()
@Test func checkoutFailsWhenCartIsEmpty()

// ❌ WRONG — test prefix is redundant and noisy
@Test func testAddsItem()
@Test func testRemove()
```

---

## Rule 3: Use init() / deinit for setup and teardown

```swift
// ✅ CORRECT — init sets up, deinit tears down
@Suite
struct DatabaseTests {
    let db: TestDatabase

    init() async throws {
        db = try await TestDatabase.create()
        try await db.seed(with: .fixtures)
    }

    deinit {
        // synchronous cleanup only
        db.close()
    }

    @Test func queryReturnsSeededUsers() async throws {
        let users = try await db.query(User.self)
        #expect(users.count == 3)
    }
}

// ❌ WRONG — XCTest lifecycle methods
override func setUp() async throws { }
override func tearDown() async throws { }
```

---

## Rule 4: Nest suites for grouping related tests

```swift
// ✅ CORRECT — nested @Suite for logical grouping
@Suite("Payment processing")
struct PaymentTests {

    @Suite("Credit card")
    struct CreditCardTests {
        @Test func acceptsVisaCard() async throws { ... }
        @Test func rejectsExpiredCard() async throws { ... }
    }

    @Suite("PayPal")
    struct PayPalTests {
        @Test func redirectsToPayPal() async throws { ... }
        @Test func handlesPayPalCancellation() async throws { ... }
    }
}
```

---

## Rule 5: Use tags to organize and filter tests

```swift
// ✅ CORRECT — declare tags in an extension
extension Tag {
    @Tag static var networking: Self
    @Tag static var authentication: Self
    @Tag static var slow: Self
}

// Apply tags to suites or individual tests
@Suite(.tags(.authentication))
struct LoginTests {
    @Test func loginWithValidCredentials() async throws { ... }

    @Test(.tags(.networking, .slow))
    func loginFetchesUserProfile() async throws { ... }
}
```

---

## Rule 6: Disable tests with `.disabled`

```swift
// ✅ CORRECT — disable with a reason
@Test(.disabled("Pending backend endpoint — ticket #1234"))
func fetchesLiveRecommendations() async throws {
    // implementation ready, waiting on API
}

// ❌ WRONG — commenting out tests loses them forever
// @Test func fetchesLiveRecommendations() async throws { ... }
```

---

## Rule 7: Control timing with `.timeLimit`

```swift
// ✅ CORRECT — time-bound slow tests
@Test(.timeLimit(.minutes(1)))
func processesLargeDataset() async throws {
    let result = try await processor.run(on: .largeFixture)
    #expect(result.count > 1000)
}
```

---

## @Suite is optional for top-level tests

If tests don't share setup, you don't need a struct or `@Suite`.
Free functions with `@Test` are valid at the top level of a test file.

```swift
// ✅ CORRECT — simple standalone tests
import Testing
@testable import MyApp

@Test func appVersionIsNotEmpty() {
    #expect(!Bundle.main.appVersion.isEmpty)
}

@Test func defaultLocaleIsSupported() {
    #expect(LocaleSupport.isSupported(.current))
}
```
