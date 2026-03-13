# Example: Async Networking Tests

Full examples for testing a URLSession-based service layer using Swift Testing.

---

## The service under test

```swift
// NetworkService.swift
protocol NetworkServiceProtocol {
    func fetch<T: Decodable>(_ type: T.Type, from endpoint: Endpoint) async throws -> T
}

final class NetworkService: NetworkServiceProtocol {
    private let session: URLSession

    init(session: URLSession = .shared) {
        self.session = session
    }

    func fetch<T: Decodable>(_ type: T.Type, from endpoint: Endpoint) async throws -> T {
        let (data, response) = try await session.data(from: endpoint.url)
        guard let http = response as? HTTPURLResponse, http.statusCode == 200 else {
            throw NetworkError.badResponse
        }
        return try JSONDecoder().decode(T.self, from: data)
    }
}
```

---

## Mock setup

```swift
// MockURLProtocol.swift — intercepts URLSession requests in tests
final class MockURLProtocol: URLProtocol {
    static var handler: ((URLRequest) throws -> (HTTPURLResponse, Data))?

    override class func canInit(with request: URLRequest) -> Bool { true }
    override class func canonicalRequest(for request: URLRequest) -> URLRequest { request }

    override func startLoading() {
        guard let handler = MockURLProtocol.handler else {
            client?.urlProtocol(self, didFailWithError: URLError(.unknown))
            return
        }
        do {
            let (response, data) = try handler(request)
            client?.urlProtocol(self, didReceive: response, cacheStoragePolicy: .notAllowed)
            client?.urlProtocol(self, didLoad: data)
            client?.urlProtocolDidFinishLoading(self)
        } catch {
            client?.urlProtocol(self, didFailWithError: error)
        }
    }

    override func stopLoading() {}
}

// Helper to create a test URLSession using the mock protocol
extension URLSession {
    static var mock: URLSession {
        let config = URLSessionConfiguration.ephemeral
        config.protocolClasses = [MockURLProtocol.self]
        return URLSession(configuration: config)
    }
}
```

---

## Full test suite

```swift
import Testing
@testable import MyApp

@Suite("NetworkService")
struct NetworkServiceTests {
    let sut: NetworkService

    init() {
        sut = NetworkService(session: .mock)
    }

    // MARK: - Success cases

    @Test func decodesUserSuccessfully() async throws {
        MockURLProtocol.handler = { _ in
            let data = try JSONEncoder().encode(User.fixture)
            let response = HTTPURLResponse(
                url: Endpoint.user("1").url,
                statusCode: 200,
                httpVersion: nil,
                headerFields: ["Content-Type": "application/json"]
            )!
            return (response, data)
        }

        let user = try await sut.fetch(User.self, from: .user("1"))
        #expect(user.id == User.fixture.id)
        #expect(user.name == User.fixture.name)
    }

    @Test func decodesListSuccessfully() async throws {
        let fixtures = [User.fixture, User.fixture2]
        MockURLProtocol.handler = { _ in
            let data = try JSONEncoder().encode(fixtures)
            let response = HTTPURLResponse(url: Endpoint.users.url, statusCode: 200, httpVersion: nil, headerFields: nil)!
            return (response, data)
        }

        let users = try await sut.fetch([User].self, from: .users)
        #expect(users.count == 2)
    }

    // MARK: - Error cases

    @Test func throwsBadResponseOnNon200() async {
        MockURLProtocol.handler = { _ in
            let response = HTTPURLResponse(url: Endpoint.user("1").url, statusCode: 404, httpVersion: nil, headerFields: nil)!
            return (response, Data())
        }

        await #expect(throws: NetworkError.badResponse) {
            try await sut.fetch(User.self, from: .user("1"))
        }
    }

    @Test func throwsDecodingErrorOnMalformedJSON() async {
        MockURLProtocol.handler = { _ in
            let response = HTTPURLResponse(url: Endpoint.user("1").url, statusCode: 200, httpVersion: nil, headerFields: nil)!
            return (response, Data("not-json".utf8))
        }

        #expect(throws: (any Error).self) {
            try await sut.fetch(User.self, from: .user("1"))
        }
    }

    @Test func throwsOnNetworkFailure() async {
        MockURLProtocol.handler = { _ in throw URLError(.notConnectedToInternet) }

        await #expect(throws: URLError.self) {
            try await sut.fetch(User.self, from: .user("1"))
        }
    }

    // MARK: - Parameterized error codes

    @Test(arguments: [400, 401, 403, 404, 500, 503])
    func throwsBadResponseForAllErrorStatusCodes(statusCode: Int) async {
        MockURLProtocol.handler = { _ in
            let response = HTTPURLResponse(url: Endpoint.user("1").url, statusCode: statusCode, httpVersion: nil, headerFields: nil)!
            return (response, Data())
        }

        await #expect(throws: NetworkError.badResponse) {
            try await sut.fetch(User.self, from: .user("1"))
        }
    }
}
```

---

## Optional: AsyncGuardKit integration

If your project uses AsyncGuardKit for structured cancellation, test guarded operations like this:

```swift
@Test func cancelsInFlightRequestOnNewCall() async throws {
    var requestCount = 0
    MockURLProtocol.handler = { _ in
        requestCount += 1
        try await Task.sleep(for: .milliseconds(200))
        let data = try JSONEncoder().encode(User.fixture)
        let response = HTTPURLResponse(url: Endpoint.user("1").url, statusCode: 200, httpVersion: nil, headerFields: nil)!
        return (response, data)
    }

    // Fire two requests in quick succession
    async let first = sut.fetch(User.self, from: .user("1"))
    async let second = sut.fetch(User.self, from: .user("2"))

    _ = try await second  // only the second should complete
    #expect(requestCount == 2)  // both were started, but first was superseded
}
```
