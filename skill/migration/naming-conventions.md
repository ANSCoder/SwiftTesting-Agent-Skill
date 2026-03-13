# Migration: XCTest Naming Conventions → Swift Testing

| Legacy XCTest naming | Swift Testing migration |
|---|---|
| `test_Validates_VisaCard` | `@Test func validatesVisaCard()` |
| `testFetchUserSuccess` | `@Test func fetchUserSuccess()` |
| `test_LoginViewModel_WhenCredentialsValid_ShouldReturnSuccess` | `@Test("login succeeds with valid credentials") func loginSucceedsWithValidCredentials()` |
| `testStep1`, `testStep2` | `@Test(arguments:)` parameterized test or clearly named separate `@Test` functions |
| Long legacy function names used for navigator readability | Keep concise function names + use `@Test("display name")` |
| `class PaymentTests: XCTestCase` naming | `@Suite("Payment") struct PaymentTests` display-name migration |

---

## 1) `test_` snake case prefix

```swift
// ❌ WRONG: keep XCTest-style prefix and separators
@Test func test_Validates_VisaCard() {
    #expect(validator.isValid(.visa))
}

// ✅ CORRECT: drop `test_`, use idiomatic lowerCamelCase
@Test func validatesVisaCard() {
    #expect(validator.isValid(.visa))
}
```

---

## 2) `testXxx` camel case prefix

```swift
// ❌ WRONG: XCTest prefix carried over
@Test func testFetchUserSuccess() async throws {
    let user = try await service.fetchUser(id: "123")
    #expect(user.id == "123")
}

// ✅ CORRECT: remove `test` prefix
@Test func fetchUserSuccess() async throws {
    let user = try await service.fetchUser(id: "123")
    #expect(user.id == "123")
}
```

---

## 3) `test_When_Should` BDD style

```swift
// ❌ WRONG: legacy BDD naming embedded in function identifier
@Test func test_LoginViewModel_WhenCredentialsValid_ShouldReturnSuccess() async {
    let result = await viewModel.login(email: "a@b.com", password: "secret")
    #expect(result == .success)
}

// ✅ CORRECT: concise function name + readable display name
@Test("login succeeds with valid credentials")
func loginSucceedsWithValidCredentials() async {
    let result = await viewModel.login(email: "a@b.com", password: "secret")
    #expect(result == .success)
}
```

---

## 4) Numbered tests (`testStep1`, `testStep2`)

```swift
// ❌ WRONG: order-dependent, numbered names with unclear intent
@Test func testStep1() {
    #expect(formatter.normalize("  visa ") == "visa")
}

@Test func testStep2() {
    #expect(formatter.normalize("MASTERCARD") == "mastercard")
}
```

```swift
// ✅ CORRECT: parameterized migration when shape is the same
@Test(arguments: ["  visa ", "MASTERCARD", "AmEx"])
func normalizesCardBrandInput(input: String) {
    #expect(!formatter.normalize(input).isEmpty)
}
```

```swift
// ✅ CORRECT: separate named tests when behaviors differ
@Test func trimsLeadingAndTrailingWhitespace() {
    #expect(formatter.normalize("  visa ") == "visa")
}

@Test func lowercasesUppercasedInput() {
    #expect(formatter.normalize("MASTERCARD") == "mastercard")
}
```

---

## 5) Keep readable navigator names with `@Test("display name")`

```swift
// ❌ WRONG: force readability only through long function names
@Test func userRegistration_WhenPasswordTooShort_ShowsValidationMessage() {
    #expect(viewModel.errorMessage == "Password too short")
}

// ✅ CORRECT: short modern function + explicit display name in navigator
@Test("user registration shows validation error for short password")
func showsValidationErrorForShortPassword() {
    #expect(viewModel.errorMessage == "Password too short")
}
```

```swift
// ✅ CORRECT: display names can preserve team wording conventions
@Test("[Checkout] card expires this month is rejected")
func rejectsCardExpiringThisMonth() {
    #expect(validator.validate(expiry: .currentMonth) == .expired)
}
```

---

## 6) Class-level naming migration with `@Suite` display names

```swift
// ❌ WRONG: rely on XCTestCase class name alone for grouping intent
final class LoginViewModelTests: XCTestCase {
    func testFetchUserSuccess() { }
}

// ✅ CORRECT: use @Suite display name for clear group labeling
@Suite("Login ViewModel")
struct LoginViewModelTests {
    @Test("fetch user succeeds")
    func fetchUserSuccess() { }
}
```

```swift
// ✅ CORRECT: preserve domain prefixes in suite labels when teams need them
@Suite("Payments / Card Validation")
struct CardValidationTests {
    @Test func validatesVisaCard() { }
}
```
