# Pattern: Assertion Messages

Swift Testing's `#expect` and `#require` accept an optional `Comment` parameter.
AI agents must include descriptive messages on non-obvious assertions.

---

## Rule: Add a message when the failure reason isn't obvious from the expression

```swift
// ✅ CORRECT — message explains the business rule, not just the code
#expect(user.isLoggedIn, "User should be authenticated after successful login")
#expect(cart.total == 0, "Cart total should reset to zero after clearing all items")
#expect(response.statusCode == 200, "API should return 200 for valid authenticated requests")

// ✅ ALSO CORRECT — simple equality with obvious meaning needs no message
#expect(items.count == 3)
#expect(user.id == "abc-123")

// ❌ WEAK — message just restates the code, adds no value
#expect(user.isLoggedIn, "isLoggedIn should be true")
#expect(items.count == 3, "count should equal 3")
```

---

## When to always include a message

**1. Boolean flags** — the flag name alone doesn't explain the expected state

```swift
// ✅ CORRECT — why should this be true right now?
#expect(viewModel.isLoading, "ViewModel should be loading immediately after load() is called")
#expect(!session.isExpired, "Session should still be valid within the expiry window")
#expect(feature.isEnabled, "Feature flag should be enabled for premium tier users")
```

**2. Computed or transformed values** — the result isn't self-evident

```swift
// ✅ CORRECT
let discounted = cart.applyDiscount(code: "SAVE20")
#expect(discounted.total == 79.99, "20% discount on $99.99 should yield $79.99")

let formatted = DateFormatter.display.string(from: date)
#expect(formatted == "March 13, 2026", "Date should format using long month name for en_US locale")
```

**3. Negative assertions** — explaining what should NOT happen

```swift
// ✅ CORRECT — why shouldn't this contain errors?
#expect(viewModel.errors.isEmpty, "No errors expected when network succeeds and data is valid")
#expect(!result.isCancelled, "Task should not be cancelled before timeout threshold is reached")
```

**4. Counts and sizes** — the expected number needs business context

```swift
// ✅ CORRECT
#expect(sections.count == 2, "Results should be grouped into Books and Courses sections")
#expect(retryAttempts == 3, "Client should retry exactly 3 times before giving up")
```

**5. Error assertions** — confirm what the error represents

```swift
// ✅ CORRECT
await #expect(
    throws: AuthError.invalidCredentials,
    "Wrong password should produce invalidCredentials, not a generic network error"
) {
    try await authService.login(email: "user@test.com", password: "wrong")
}
```

---

## Parameterized test messages

In parameterized tests, Swift Testing automatically includes the argument value
in failure output. You still benefit from a message that describes the rule:

```swift
@Test(arguments: ["", " ", "\t", "\n"])
func rejectsBlankUsernames(username: String) {
    #expect(
        !validator.isValid(username),
        "Blank or whitespace-only usernames should always fail validation"
    )
}
```

---

## #require messages

```swift
// ✅ CORRECT — message explains what this value should be present
let token = try #require(
    keychain.read(key: "auth_token"),
    "Auth token should exist in keychain after successful login"
)
```

---

## Before / After — the impact of messages

```swift
// ❌ BEFORE — agent default, no context on failure
#expect(order.status == .confirmed)
#expect(order.items.count == 2)
#expect(!order.isPending)

// ✅ AFTER — with this skill applied
#expect(order.status == .confirmed, "Order should be confirmed immediately after payment succeeds")
#expect(order.items.count == 2, "Order should preserve both line items from the cart")
#expect(!order.isPending, "Confirmed order should no longer be in pending state")
```

When these tests fail in CI, the message tells you exactly which business rule broke —
not just which line of code failed.

---

## Summary: message decision rule

Ask: **"If this test fails at 2am, will the message tell me which business rule broke?"**

- Yes → message is good
- No → rewrite the message or add one
- The expression makes it obvious → skip the message
