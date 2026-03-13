# Example: Delegate Method Testing

Testing delegate-based APIs using Swift Testing.
Covers URLSessionDelegate, CLLocationManagerDelegate, and custom delegates.

The key pattern throughout: `confirmation {}` replaces `XCTestExpectation`.

---

## URLSessionDelegate

### Testing upload progress

```swift
import Testing
@testable import MyApp

// Spy delegate — captures delegate calls
final class UploadDelegateSpy: NSObject, URLSessionTaskDelegate {
    var progressValues: [Double] = []
    var didComplete = false

    func urlSession(
        _ session: URLSession,
        task: URLSessionTask,
        didSendBodyData bytesSent: Int64,
        totalBytesSent: Int64,
        totalBytesExpectedToSend: Int64
    ) {
        let progress = Double(totalBytesSent) / Double(totalBytesExpectedToSend)
        progressValues.append(progress)
    }

    func urlSession(_ session: URLSession, task: URLSessionTask, didCompleteWithError error: Error?) {
        didComplete = true
    }
}

@Suite("URLSession upload delegate")
struct URLSessionUploadDelegateTests {

    @Test func reportsProgressDuringUpload() async throws {
        let spy = UploadDelegateSpy()
        let session = URLSession(configuration: .ephemeral, delegate: spy, delegateQueue: nil)
        let service = UploadService(session: session)

        await confirmation("upload completes", expectedCount: 1) { confirm in
            Task {
                try await service.upload(data: Data(repeating: 0, count: 1024 * 100))
                confirm()
            }
        }

        #expect(!spy.progressValues.isEmpty, "Upload should report at least one progress update")
        #expect(spy.progressValues.last == 1.0, "Final progress value should be 1.0 (100%)")
        #expect(spy.didComplete, "Delegate should receive didComplete after upload finishes")
    }

    @Test func progressValuesAreMonotonicallyIncreasing() async throws {
        let spy = UploadDelegateSpy()
        let session = URLSession(configuration: .ephemeral, delegate: spy, delegateQueue: nil)
        let service = UploadService(session: session)

        try await service.upload(data: Data(repeating: 0, count: 1024 * 50))

        let isMonotonic = zip(spy.progressValues, spy.progressValues.dropFirst())
            .allSatisfy { $0 <= $1 }
        #expect(isMonotonic, "Progress should never decrease during an upload")
    }
}
```

### Testing URLSessionDataDelegate (streaming response)

```swift
final class StreamingDelegateSpy: NSObject, URLSessionDataDelegate {
    var receivedChunks: [Data] = []
    var didFinish = false

    func urlSession(_ session: URLSession, dataTask: URLSessionDataTask, didReceive data: Data) {
        receivedChunks.append(data)
    }

    func urlSession(_ session: URLSession, task: URLSessionTask, didCompleteWithError error: Error?) {
        if error == nil { didFinish = true }
    }
}

@Suite("URLSession streaming delegate")
struct URLSessionStreamingTests {

    @Test func receivesDataInChunks() async throws {
        let spy = StreamingDelegateSpy()
        let session = URLSession(configuration: .ephemeral, delegate: spy, delegateQueue: nil)
        let client = StreamingClient(session: session)

        await confirmation("streaming completes") { confirm in
            Task {
                try await client.stream(endpoint: .largeResponse)
                confirm()
            }
        }

        #expect(spy.receivedChunks.count > 1, "Large response should arrive in multiple chunks")
        #expect(spy.didFinish, "Delegate should signal completion after all chunks received")

        let totalBytes = spy.receivedChunks.reduce(0) { $0 + $1.count }
        #expect(totalBytes > 0, "Total received bytes should be non-zero")
    }
}
```

---

## CLLocationManagerDelegate

### Testing location authorization flow

```swift
import CoreLocation

// Mock location manager — avoids requiring real device permissions
final class MockLocationManager: CLLocationManager {
    var stubbedAuthorizationStatus: CLAuthorizationStatus = .notDetermined

    override var authorizationStatus: CLAuthorizationStatus {
        stubbedAuthorizationStatus
    }

    var requestWhenInUseAuthorizationCalled = false
    override func requestWhenInUseAuthorization() {
        requestWhenInUseAuthorizationCalled = true
        // Simulate the delegate callback
        delegate?.locationManagerDidChangeAuthorization?(self)
    }
}

final class LocationDelegateSpy: NSObject, CLLocationManagerDelegate {
    var authorizationChanges: [CLAuthorizationStatus] = []
    var receivedLocations: [CLLocation] = []
    var receivedError: Error?

    func locationManagerDidChangeAuthorization(_ manager: CLLocationManager) {
        authorizationChanges.append(manager.authorizationStatus)
    }

    func locationManager(_ manager: CLLocationManager, didUpdateLocations locations: [CLLocation]) {
        receivedLocations.append(contentsOf: locations)
    }

    func locationManager(_ manager: CLLocationManager, didFailWithError error: Error) {
        receivedError = error
    }
}

@Suite("Location service delegate")
struct LocationServiceDelegateTests {

    @Test func requestsAuthorizationOnFirstUse() async throws {
        let mockManager = MockLocationManager()
        let spy = LocationDelegateSpy()
        mockManager.delegate = spy
        let service = LocationService(manager: mockManager)

        await confirmation("authorization requested") { confirm in
            mockManager.stubbedAuthorizationStatus = .authorizedWhenInUse
            service.startTracking()
            confirm()
        }

        #expect(
            mockManager.requestWhenInUseAuthorizationCalled,
            "Location service should request authorization before starting tracking"
        )
    }

    @Test func delegateReceivesAuthorizationChange() async {
        let mockManager = MockLocationManager()
        let spy = LocationDelegateSpy()
        mockManager.delegate = spy

        await confirmation("authorization change received") { confirm in
            mockManager.stubbedAuthorizationStatus = .authorizedAlways
            // Trigger the delegate callback
            spy.locationManagerDidChangeAuthorization(mockManager)
            confirm()
        }

        #expect(
            spy.authorizationChanges.contains(.authorizedAlways),
            "Delegate should record the authorizedAlways status change"
        )
    }

    @Test func delegateReceivesLocationUpdates() async throws {
        let mockManager = MockLocationManager()
        let spy = LocationDelegateSpy()
        mockManager.delegate = spy
        let service = LocationService(manager: mockManager)
        let expectedLocation = CLLocation(latitude: 37.7749, longitude: -122.4194)

        await confirmation("location update received") { confirm in
            service.onLocationReceived = { _ in confirm() }
            spy.locationManager(mockManager, didUpdateLocations: [expectedLocation])
        }

        let received = try #require(spy.receivedLocations.first, "Should have received at least one location")
        #expect(received.coordinate.latitude == 37.7749, "Latitude should match the stubbed location")
        #expect(received.coordinate.longitude == -122.4194, "Longitude should match the stubbed location")
    }

    @Test func delegateHandlesLocationError() async {
        let mockManager = MockLocationManager()
        let spy = LocationDelegateSpy()
        mockManager.delegate = spy

        let error = CLError(.denied)
        spy.locationManager(mockManager, didFailWithError: error)

        #expect(spy.receivedError != nil, "Delegate should capture location errors")
        let clError = try #require(spy.receivedError as? CLError)
        #expect(clError.code == .denied, "Error code should be .denied when user revokes permission")
    }

    @Test(arguments: [
        CLAuthorizationStatus.denied,
        .restricted,
        .notDetermined,
    ])
    func stopsTrackingForNonAuthorizedStatuses(status: CLAuthorizationStatus) {
        let mockManager = MockLocationManager()
        mockManager.stubbedAuthorizationStatus = status
        let service = LocationService(manager: mockManager)

        service.startTracking()

        #expect(
            !service.isTracking,
            "Location tracking should not start when authorization status is \(status)"
        )
    }
}
```

---

## Custom delegate pattern

### Testing a download manager delegate

```swift
protocol DownloadManagerDelegate: AnyObject {
    func downloadManager(_ manager: DownloadManager, didStartDownload id: String)
    func downloadManager(_ manager: DownloadManager, didUpdateProgress progress: Double, for id: String)
    func downloadManager(_ manager: DownloadManager, didFinishDownload id: String, to url: URL)
    func downloadManager(_ manager: DownloadManager, didFailDownload id: String, with error: Error)
}

// Spy captures all delegate calls for assertion
final class DownloadDelegateSpy: DownloadManagerDelegate {
    var startedIDs: [String] = []
    var progressUpdates: [(id: String, progress: Double)] = []
    var finishedDownloads: [(id: String, url: URL)] = []
    var failedDownloads: [(id: String, error: Error)] = []

    func downloadManager(_ manager: DownloadManager, didStartDownload id: String) {
        startedIDs.append(id)
    }

    func downloadManager(_ manager: DownloadManager, didUpdateProgress progress: Double, for id: String) {
        progressUpdates.append((id: id, progress: progress))
    }

    func downloadManager(_ manager: DownloadManager, didFinishDownload id: String, to url: URL) {
        finishedDownloads.append((id: id, url: url))
    }

    func downloadManager(_ manager: DownloadManager, didFailDownload id: String, with error: Error) {
        failedDownloads.append((id: id, error: error))
    }
}

@Suite("DownloadManager delegate")
struct DownloadManagerDelegateTests {
    let spy: DownloadDelegateSpy
    let manager: DownloadManager

    init() {
        spy = DownloadDelegateSpy()
        manager = DownloadManager()
        manager.delegate = spy
    }

    @Test func notifiesDelegateWhenDownloadStarts() async throws {
        await confirmation("download started") { confirm in
            manager.onDidStart = { _ in confirm() }
            manager.startDownload(id: "file-001", url: URL.fixture)
        }

        #expect(spy.startedIDs.contains("file-001"), "Delegate should be notified when download starts")
    }

    @Test func notifiesDelegateOnCompletion() async throws {
        await confirmation("download finished") { confirm in
            manager.onDidFinish = { _, _ in confirm() }
            manager.startDownload(id: "file-002", url: URL.fixture)
        }

        let finished = try #require(
            spy.finishedDownloads.first { $0.id == "file-002" },
            "Delegate should receive didFinishDownload for file-002"
        )
        #expect(finished.url.isFileURL, "Finished download URL should be a local file URL")
    }

    @Test func notifiesDelegateOnFailure() async {
        await confirmation("download failed") { confirm in
            manager.onDidFail = { _, _ in confirm() }
            manager.startDownload(id: "file-bad", url: URL.invalid)
        }

        #expect(
            spy.failedDownloads.first { $0.id == "file-bad" } != nil,
            "Delegate should receive didFailDownload for invalid URLs"
        )
        #expect(spy.finishedDownloads.isEmpty, "Failed download should not trigger didFinishDownload")
    }

    @Test func delegateReceivesProgressInOrder() async throws {
        await confirmation("download completes") { confirm in
            manager.onDidFinish = { _, _ in confirm() }
            manager.startDownload(id: "file-003", url: URL.fixture)
        }

        let updates = spy.progressUpdates.filter { $0.id == "file-003" }.map { $0.progress }
        let isMonotonic = zip(updates, updates.dropFirst()).allSatisfy { $0 <= $1 }
        #expect(isMonotonic, "Progress updates should only increase for a single download")
        #expect(updates.last == 1.0, "Final progress update should be 1.0")
    }

    @Test func supportsMultipleConcurrentDownloads() async throws {
        await confirmation("both downloads complete", expectedCount: 2) { confirm in
            manager.onDidFinish = { _, _ in confirm() }
            manager.startDownload(id: "concurrent-A", url: URL.fixture)
            manager.startDownload(id: "concurrent-B", url: URL.fixture2)
        }

        #expect(spy.finishedDownloads.count == 2, "Both concurrent downloads should complete")
        #expect(
            spy.startedIDs.contains("concurrent-A") && spy.startedIDs.contains("concurrent-B"),
            "Both downloads should have been started"
        )
    }
}
```

---

## SwiftUI + delegate interop

When a SwiftUI feature uses a UIKit delegate under the hood (via `UIViewRepresentable` or a coordinator):

```swift
@Suite("MapView coordinator delegate")
@MainActor
struct MapViewCoordinatorTests {
    let coordinator: MapViewCoordinator
    let viewModel: MapViewModel

    init() {
        viewModel = MapViewModel()
        coordinator = MapViewCoordinator(viewModel: viewModel)
    }

    @Test func coordinatorForwardsAnnotationTapToViewModel() {
        let annotation = MKPointAnnotation()
        annotation.title = "Test Pin"

        coordinator.mapView(MKMapView(), didSelect: MKAnnotationView())

        #expect(
            viewModel.selectedAnnotation != nil,
            "Coordinator should forward annotation selection to ViewModel"
        )
    }
}
```

---

## Summary

| Delegate type | Pattern |
|---|---|
| URLSessionTaskDelegate / DataDelegate | Spy class + `confirmation {}` |
| CLLocationManagerDelegate | Mock CLLocationManager + spy delegate |
| Custom protocol delegate | Spy class capturing all calls |
| SwiftUI UIViewRepresentable coordinator | Test the coordinator directly with a real or mock mapView |
| Multiple events expected | `confirmation("label", expectedCount: N)` |
| Ordering / monotonicity | Collect values in spy, assert with `allSatisfy` after confirmation |
