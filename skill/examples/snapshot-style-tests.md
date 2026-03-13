# Example: Structural Output Tests

Testing the shape and structure of output — without UI snapshot dependencies.
These replace pixel-based snapshot tests for business logic and data transformation.

---

## Testing data transformations

```swift
import Testing
@testable import MyApp

@Suite("Order summary formatting")
struct OrderSummaryFormatterTests {

    let formatter = OrderSummaryFormatter()

    @Test func formatsSingleItemOrder() {
        let order = Order(items: [.init(name: "Widget", price: 9.99, quantity: 1)])
        let summary = formatter.format(order)

        #expect(summary.lineItems.count == 1)
        #expect(summary.lineItems[0].label == "Widget")
        #expect(summary.lineItems[0].value == "$9.99")
        #expect(summary.totalLabel == "Total")
        #expect(summary.totalValue == "$9.99")
    }

    @Test func formatsMultiItemOrderWithDiscount() {
        let order = Order(
            items: [
                .init(name: "Widget", price: 9.99, quantity: 2),
                .init(name: "Gadget", price: 24.99, quantity: 1),
            ],
            discountCode: "SAVE10"
        )
        let summary = formatter.format(order)

        #expect(summary.lineItems.count == 3)  // 2 items + discount line
        let discountLine = try #require(summary.lineItems.first { $0.isDiscount })
        #expect(discountLine.value.hasPrefix("-"))  // discount shows as negative
        #expect(summary.totalValue == "$39.58")   // (9.99×2 + 24.99) × 0.90
    }

    @Test(arguments: [
        (Locale(identifier: "en_US"), "$", "."),
        (Locale(identifier: "de_DE"), "€", ","),
        (Locale(identifier: "ja_JP"), "¥", "."),
    ])
    func formatsCorrectlyForLocale(locale: Locale, currencySymbol: String, decimalSeparator: String) {
        let order = Order(items: [.init(name: "Item", price: 10.50, quantity: 1)])
        let summary = formatter.format(order, locale: locale)
        #expect(summary.totalValue.contains(currencySymbol))
    }
}
```

---

## Testing serialization / JSON output shape

```swift
@Suite("API request encoding")
struct APIRequestEncoderTests {

    let encoder = APIRequestEncoder()

    @Test func encodesPaginationParameters() throws {
        let request = APIRequest.listUsers(page: 2, perPage: 25)
        let encoded = try encoder.encode(request)
        let json = try #require(try JSONSerialization.jsonObject(with: encoded) as? [String: Any])

        #expect(json["page"] as? Int == 2)
        #expect(json["per_page"] as? Int == 25)
        #expect(json["page"] as? Int != 0)  // page is 1-indexed
    }

    @Test func omitsNilFieldsFromEncoding() throws {
        let request = APIRequest.updateUser(id: "123", name: "Alice", email: nil)
        let encoded = try encoder.encode(request)
        let json = try #require(try JSONSerialization.jsonObject(with: encoded) as? [String: Any])

        #expect(json["name"] as? String == "Alice")
        #expect(json["email"] == nil)  // nil fields must not appear in output
    }

    @Test func includesAuthHeaderWhenTokenPresent() throws {
        let request = APIRequest.listUsers(page: 1, perPage: 10)
        let urlRequest = try encoder.buildURLRequest(for: request, token: "tok_abc123")

        let authHeader = try #require(urlRequest.value(forHTTPHeaderField: "Authorization"))
        #expect(authHeader == "Bearer tok_abc123")
    }
}
```

---

## Testing state machine transitions

```swift
@Suite("Download state machine")
struct DownloadStateMachineTests {

    @Test(arguments: [
        (DownloadState.idle, DownloadEvent.start, DownloadState.downloading),
        (DownloadState.downloading, DownloadEvent.pause, DownloadState.paused),
        (DownloadState.paused, DownloadEvent.resume, DownloadState.downloading),
        (DownloadState.downloading, DownloadEvent.complete, DownloadState.complete),
        (DownloadState.downloading, DownloadEvent.fail, DownloadState.failed),
    ])
    func transitionsToCorrectState(
        from initial: DownloadState,
        on event: DownloadEvent,
        expected: DownloadState
    ) {
        let machine = DownloadStateMachine(initial: initial)
        machine.handle(event)
        #expect(machine.state == expected)
    }

    @Test(arguments: [
        (DownloadState.complete, DownloadEvent.start),
        (DownloadState.failed, DownloadEvent.pause),
        (DownloadState.idle, DownloadEvent.complete),
    ])
    func ignoresInvalidTransitions(from state: DownloadState, on event: DownloadEvent) {
        let machine = DownloadStateMachine(initial: state)
        machine.handle(event)
        #expect(machine.state == state)  // unchanged
    }
}
```

---

## Testing view model output shape (without SwiftUI)

```swift
@Suite("Search results view model")
@MainActor
struct SearchResultsViewModelTests {

    let sut: SearchResultsViewModel

    init() {
        sut = SearchResultsViewModel(service: MockSearchService())
    }

    @Test func emptyQueryShowsNoResults() async {
        await sut.search(query: "")
        #expect(sut.sections.isEmpty)
        #expect(sut.emptyStateMessage != nil)
    }

    @Test func resultsGroupedByCategory() async {
        let mockService = sut.service as! MockSearchService
        mockService.results = [
            .init(title: "Swift book", category: .books),
            .init(title: "Swift course", category: .courses),
            .init(title: "Advanced Swift", category: .books),
        ]

        await sut.search(query: "swift")

        #expect(sut.sections.count == 2)
        let bookSection = try #require(sut.sections.first { $0.title == "Books" })
        #expect(bookSection.items.count == 2)
    }

    @Test func highlightsQueryTermInResultTitles() async {
        let mockService = sut.service as! MockSearchService
        mockService.results = [.init(title: "Swift Testing Guide", category: .books)]
        await sut.search(query: "Swift")

        let item = try #require(sut.sections.first?.items.first)
        #expect(item.highlightedTitle.contains("Swift"))
    }
}
```
