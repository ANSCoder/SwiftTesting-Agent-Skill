# Pattern: Parameterized Tests

Swift Testing has native parameterized test support. Never generate repeated test methods for multiple inputs.

---

## Rule 1: Use `@Test(arguments:)` instead of repeating test methods

```swift
// ✅ CORRECT — one test, many inputs
@Test(arguments: ["alice@example.com", "bob@work.io", "user+tag@domain.co.uk"])
func validEmailsPassValidation(email: String) {
    #expect(EmailValidator.isValid(email))
}

// ❌ WRONG — never repeat tests for different inputs
@Test func validatesAlice() { #expect(EmailValidator.isValid("alice@example.com")) }
@Test func validatesBob() { #expect(EmailValidator.isValid("bob@work.io")) }
@Test func validatesTagEmail() { #expect(EmailValidator.isValid("user+tag@domain.co.uk")) }
```

---

## Rule 2: Parameterize with enums for readability

```swift
// ✅ CORRECT — enums make test cases self-documenting
enum AuthScenario: CaseIterable {
    case validCredentials, expiredToken, wrongPassword, lockedAccount
}

@Test(arguments: AuthScenario.allCases)
func authenticationHandlesAllScenarios(scenario: AuthScenario) async throws {
    let credentials = Credentials.fixture(for: scenario)
    let expectedResult = AuthResult.expected(for: scenario)
    let result = try await authService.authenticate(with: credentials)
    #expect(result == expectedResult)
}
```

---

## Rule 3: Parameterize with tuples for input/output pairs

```swift
// ✅ CORRECT
@Test(arguments: [
    ("hello world", "Hello World"),
    ("swift testing", "Swift Testing"),
    ("ios developer", "Ios Developer"),
])
func titleCaseConvertsCorrectly(input: String, expected: String) {
    #expect(input.toTitleCase() == expected)
}

// ✅ CORRECT — named tuple fields for clarity
@Test(arguments: zip(
    [1, 2, 3, 4, 5],
    [1, 4, 9, 16, 25]
))
func squaresAreCorrect(input: Int, expected: Int) {
    #expect(Math.square(input) == expected)
}
```

---

## Rule 4: Combine two argument collections for matrix testing

```swift
// ✅ CORRECT — tests every combination of region × currency
@Test(arguments: Region.allCases, Currency.allCases)
func priceFormatsCorrectlyForAllRegions(region: Region, currency: Currency) {
    let formatted = PriceFormatter.format(100.0, region: region, currency: currency)
    #expect(!formatted.isEmpty)
    #expect(formatted.contains(currency.symbol))
}
```

---

## Rule 5: Use `CustomTestStringConvertible` for clear failure messages

When a test fails, Swift Testing shows the argument value. For custom types,
conform to `CustomTestStringConvertible` so failure messages are readable.

```swift
// ✅ CORRECT
struct TestUser: CustomTestStringConvertible {
    let id: String
    let role: Role
    
    var testDescription: String {
        "User(id: \(id), role: \(role))"
    }
}

@Test(arguments: [
    TestUser(id: "admin-1", role: .admin),
    TestUser(id: "viewer-2", role: .viewer),
])
func accessControlRespectedForAllRoles(user: TestUser) async throws {
    let canEdit = try await permissionService.canEdit(userID: user.id)
    #expect(canEdit == (user.role == .admin))
}
```

---

## Rule 6: Async parameterized tests

```swift
// ✅ CORRECT — parameterized tests can be async throws
@Test(arguments: ["user-1", "user-2", "user-3"])
func fetchesEachUserSuccessfully(userID: String) async throws {
    let user = try await userService.fetch(id: userID)
    #expect(user.id == userID)
}
```

---

## Summary

| Scenario | Pattern |
|---|---|
| Multiple inputs, same logic | `@Test(arguments: [...]) func name(param: T)` |
| Enum-driven scenarios | `@Test(arguments: MyEnum.allCases)` |
| Input/output pairs | `@Test(arguments: zip(inputs, expectedOutputs))` |
| Full matrix (all combos) | `@Test(arguments: collectionA, collectionB)` |
| Readable failure messages | Conform argument type to `CustomTestStringConvertible` |
