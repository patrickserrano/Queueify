# ADR-005: Concurrency Architecture — Actors, Structured Concurrency, and Background Tasks

**Status:** Accepted
**Date:** 2026-03-31
**Deciders:** Queueify Engineering

## Context

Queueify performs multiple concurrent operations that must be coordinated safely: capturing tracks from Spotify and Apple Music, deduplicating entries, persisting to SwiftData, syncing with the API server (Pro tier), and handling app lifecycle transitions between foreground and background. The app targets iOS 26+ with Swift 6 strict concurrency checking enabled. All data races must be caught at compile time.

Key concurrency requirements:

- **Multiple capture sessions.** Free-tier users may run up to 3 concurrent capture sessions. Each session involves an independent polling or observation loop, deduplication state, and SwiftData writes.
- **Heterogeneous capture services.** Spotify capture is driven by a recurring `Task.sleep`-based polling loop. Apple Music capture is driven by an `AsyncSequence` observation of MusicKit's player state. Both must be startable, stoppable, and cancellable independently.
- **Persistence.** SwiftData `ModelContext` is not `Sendable` and must be confined to a single actor. Writes from multiple capture sessions must be serialized.
- **Background execution.** `BGTaskScheduler` fires Background App Refresh tasks that run backfill and sync work in a ~30-second window. These tasks must be cancellable via the system's expiration handler.
- **Network sync.** Pro users sync playlists bidirectionally with the Queueify API server. Sync must handle network failures gracefully, retry with exponential backoff, and never block capture.
- **UI updates.** The SwiftUI layer observes state via the `@Observable` macro. All UI-bound state must be isolated to `@MainActor`.

The previous generation of the codebase used GCD queues and Combine pipelines. These are no longer acceptable under the project's Swift 6 strict concurrency rules and have been removed entirely.

## Decision

Adopt **Swift Concurrency as the sole concurrency model**: `actor` isolation for mutable state, structured concurrency (`TaskGroup`, child tasks) for bounded work, and unstructured `Task` only where structured concurrency cannot express the required lifetime (e.g., long-running capture loops tied to user intent rather than a lexical scope). All types crossing actor boundaries must be `Sendable`. No Combine, no GCD, no completion handlers, no singletons.

## Options Considered

### Option A: GCD + Combine (legacy)

Continue using Dispatch queues for serialization and Combine publishers for reactive observation. Wrap Combine publishers in `AsyncSequence` bridges where needed.

**Rejected.** Swift 6 strict concurrency checking does not understand GCD queue isolation. `DispatchQueue`-based synchronization produces false-positive data-race warnings that cannot be silenced without `@unchecked Sendable`, which defeats the purpose of strict checking. Combine's `AnyCancellable` bags are not `Sendable` and create ownership hazards across actor boundaries. Maintaining two concurrency worlds (Combine + async/await) doubles the cognitive overhead and bug surface area.

### Option B: Swift Concurrency (actors + structured concurrency)

Use `actor` types for all mutable shared state. Use `async/await` for all asynchronous calls. Use `TaskGroup` for bounded concurrent work (e.g., parallel backfill of multiple sessions). Use `AsyncSequence` and `AsyncStream` for event-driven observation. Use `@MainActor`-isolated `@Observable` classes for UI state. Use `ModelActor` for SwiftData persistence.

**Accepted.** This option provides compile-time data-race safety under Swift 6 strict concurrency, eliminates the Combine/GCD bridge layer, and aligns with Apple's platform direction for iOS 26+.

## Rationale

1. **Compile-time safety.** Actor isolation boundaries are enforced by the compiler. Every cross-actor call is `await`-ed, making suspension points visible. `Sendable` checking prevents accidental sharing of non-thread-safe types.
2. **Cancellation is structural.** `Task.isCancelled` and `CancellationError` propagate automatically through child tasks and `TaskGroup`. When a user stops a capture session, cancelling the parent task tears down polling loops, observation streams, and in-flight network calls without manual bookkeeping.
3. **Backpressure via AsyncStream.** `AsyncStream` with a bounded buffer policy (`.bufferingNewest` or `.bufferingOldest`) provides natural backpressure between the capture layer and the persistence layer, preventing unbounded memory growth during burst playback scenarios.
4. **ModelActor alignment.** SwiftData's `ModelActor` protocol creates a dedicated actor with its own `ModelContext`, which is exactly the isolation model we need. Capture actors send `Sendable` value types (DTOs) to the `ModelActor`, which performs the actual insert/update.
5. **BGTaskScheduler compatibility.** `BGTaskScheduler` completion handlers run on arbitrary threads but can trivially `await` actor-isolated methods, bridging into the structured concurrency world.

## Consequences

### Positive

- All data races are caught at compile time with Swift 6 strict concurrency enabled. No runtime thread sanitizer surprises.
- Cancellation propagation means stopping a capture session is a single `task.cancel()` call; all downstream work (polling timer, network requests, SwiftData writes) tears down automatically.
- Actor reentrancy is explicit and auditable. Suspension points are marked with `await`, making it straightforward to reason about interleaving.
- The architecture naturally supports the free-tier limit of 3 concurrent sessions: `CaptureEngine` tracks active `Task` handles in a dictionary and rejects new sessions when the count reaches the cap.

### Negative

- Actor reentrancy requires careful design. A long-running `await` inside an actor method allows other calls to interleave. State must be captured before `await` and validated after to avoid TOCTOU bugs.
- `Sendable` conformance is viral. Every type that crosses an actor boundary must be `Sendable`, which rules out many legacy Foundation types and forces the use of value-type DTOs at isolation boundaries.
- Debugging structured concurrency is harder than debugging GCD. Swift concurrency instruments are improving but stack traces through `Task` and `TaskGroup` can be less legible than Dispatch queue labels.

### Risks

- **Actor contention on CaptureEngine.** If multiple capture sessions send high-frequency updates to a single `CaptureEngine` actor, the actor's serial executor could become a bottleneck. Mitigation: each session writes to the `ModelActor` directly for persistence; `CaptureEngine` only coordinates lifecycle and deduplication metadata.
- **Background task expiration.** `BGTaskScheduler` may terminate work before sync completes. Mitigation: use `withTaskCancellationHandler` to checkpoint progress when the system calls the expiration handler.
- **MusicKit AsyncSequence contract changes.** Apple may change the observation API surface in future SDK releases. Mitigation: isolate MusicKit access behind the `CaptureService` protocol so the concrete implementation can be swapped without affecting the actor hierarchy.

## Implementation Notes

### Actor Hierarchy

The following pseudocode illustrates the actor isolation boundaries and task lifecycle:

```swift
// MARK: - Capture Service Protocol

protocol CaptureService: Actor, Sendable {
    func start(session: CaptureSession.ID) async throws
    func stop() async
    var trackStream: AsyncStream<CapturedTrack> { get }
}

// MARK: - Spotify Capture (polling loop via unstructured Task)

actor SpotifyCaptureService: CaptureService {
    private var pollingTask: Task<Void, Never>?
    private let continuation: AsyncStream<CapturedTrack>.Continuation
    let trackStream: AsyncStream<CapturedTrack>

    func start(session: CaptureSession.ID) async throws {
        pollingTask = Task {
            while !Task.isCancelled {
                do {
                    let track = try await SpotifyAPI.currentlyPlaying()
                    continuation.yield(track)
                    try await Task.sleep(for: .seconds(adaptiveInterval))
                } catch is CancellationError {
                    break
                } catch {
                    try? await Task.sleep(for: .seconds(backoffInterval))
                }
            }
            continuation.finish()
        }
    }

    func stop() async {
        pollingTask?.cancel()
        pollingTask = nil
    }
}

// MARK: - Apple Music Capture (AsyncSequence observation)

actor MusicKitCaptureService: CaptureService {
    private var observationTask: Task<Void, Never>?
    private let continuation: AsyncStream<CapturedTrack>.Continuation
    let trackStream: AsyncStream<CapturedTrack>

    func start(session: CaptureSession.ID) async throws {
        observationTask = Task {
            let player = ApplicationMusicPlayer.shared
            for await entry in player.queue.currentEntry.values {
                guard !Task.isCancelled else { break }
                if let track = CapturedTrack(from: entry) {
                    continuation.yield(track)
                }
            }
            continuation.finish()
        }
    }

    func stop() async {
        observationTask?.cancel()
        observationTask = nil
    }
}

// MARK: - Persistence via ModelActor

@ModelActor
actor PersistenceActor {
    func insertTrack(_ dto: CapturedTrackDTO, session: CaptureSession.ID) throws {
        let track = CapturedTrackRecord(from: dto, sessionID: session)
        modelContext.insert(track)
        try modelContext.save()
    }

    func insertTracks(_ dtos: [CapturedTrackDTO], session: CaptureSession.ID) throws {
        for dto in dtos {
            let track = CapturedTrackRecord(from: dto, sessionID: session)
            modelContext.insert(track)
        }
        try modelContext.save()
    }
}

// MARK: - CaptureEngine (coordinator actor)

actor CaptureEngine {
    private var activeSessions: [CaptureSession.ID: SessionHandle] = [:]
    private let persistence: PersistenceActor
    private let maxConcurrentSessions: Int  // 3 for free, unlimited for Pro

    struct SessionHandle: Sendable {
        let service: any CaptureService
        let consumptionTask: Task<Void, Never>
        let sessionID: CaptureSession.ID
    }

    func startSession(
        id: CaptureSession.ID,
        service: any CaptureService
    ) async throws {
        guard activeSessions.count < maxConcurrentSessions else {
            throw CaptureError.sessionLimitReached
        }

        try await service.start(session: id)

        // Structured consumption: read from the service's stream,
        // deduplicate, and write to SwiftData.
        let consumptionTask = Task {
            var seen = Set<TrackFingerprint>()
            for await track in await service.trackStream {
                guard !Task.isCancelled else { break }
                let fingerprint = TrackFingerprint(track)
                guard seen.insert(fingerprint).inserted else { continue }
                do {
                    try await persistence.insertTrack(
                        CapturedTrackDTO(track),
                        session: id
                    )
                } catch {
                    // Log persistence error; do not stop capture.
                }
            }
        }

        activeSessions[id] = SessionHandle(
            service: service,
            consumptionTask: consumptionTask,
            sessionID: id
        )
    }

    func stopSession(id: CaptureSession.ID) async {
        guard let handle = activeSessions.removeValue(forKey: id) else { return }
        await handle.service.stop()
        handle.consumptionTask.cancel()
    }

    func stopAll() async {
        for id in activeSessions.keys {
            await stopSession(id: id)
        }
    }
}

// MARK: - SyncEngine (Pro tier, bidirectional sync)

actor SyncEngine {
    private var syncTask: Task<Void, Never>?

    func startPeriodicSync() {
        syncTask = Task {
            while !Task.isCancelled {
                do {
                    try await performSync()
                    try await Task.sleep(for: .minutes(5))
                } catch is CancellationError {
                    break
                } catch {
                    try? await Task.sleep(for: .seconds(retryBackoff()))
                }
            }
        }
    }

    private func performSync() async throws {
        try await withThrowingTaskGroup(of: Void.self) { group in
            group.addTask { try await self.pushLocalChanges() }
            group.addTask { try await self.pullRemoteChanges() }
            try await group.waitForAll()
        }
    }

    func cancel() {
        syncTask?.cancel()
        syncTask = nil
    }
}

// MARK: - UI Layer (@MainActor + @Observable)

@MainActor
@Observable
final class CaptureViewModel {
    private(set) var activeSessions: [CaptureSessionViewState] = []
    private(set) var isCapturing = false

    private let engine: CaptureEngine

    func startSession(service: any CaptureService) async {
        let id = CaptureSession.ID()
        do {
            try await engine.startSession(id: id, service: service)
            isCapturing = true
            // Refresh view state from persistence layer.
            await refreshActiveSessions()
        } catch CaptureError.sessionLimitReached {
            // Surface upgrade prompt.
        } catch {
            // Surface error in UI.
        }
    }

    func stopSession(id: CaptureSession.ID) async {
        await engine.stopSession(id: id)
        await refreshActiveSessions()
        isCapturing = !activeSessions.isEmpty
    }
}

// MARK: - Background App Refresh Integration

enum BackgroundTaskCoordinator {
    static func registerTasks() {
        BGTaskScheduler.shared.register(
            forTaskWithIdentifier: "com.queueify.backfill",
            using: nil
        ) { bgTask in
            let task = Task {
                await withTaskCancellationHandler {
                    do {
                        try await performBackfill()
                        bgTask.setTaskCompleted(success: true)
                    } catch {
                        bgTask.setTaskCompleted(success: false)
                    }
                } onCancel: {
                    bgTask.setTaskCompleted(success: false)
                }
            }

            bgTask.expirationHandler = {
                task.cancel()
            }
        }
    }

    private static func performBackfill() async throws {
        try await withThrowingTaskGroup(of: Void.self) { group in
            group.addTask { try await SpotifyAPI.backfillRecentlyPlayed() }
            group.addTask { try await MusicKitAPI.backfillRecentlyPlayed() }
            try await group.waitForAll()
        }
    }
}
```

### Key Design Rules

1. **Actor isolation boundaries.** `CaptureEngine`, `SpotifyCaptureService`, `MusicKitCaptureService`, `SyncEngine`, and `PersistenceActor` are each isolated actors. Communication between them is always `await`-ed. Data crossing boundaries must be `Sendable` value types (structs, enums) or other actors.

2. **Sendable conformance.** `CapturedTrack`, `CapturedTrackDTO`, `TrackFingerprint`, and `CaptureSession.ID` are all `Sendable` structs. No classes cross actor boundaries unless they are actors themselves or `@unchecked Sendable` with documented justification (none expected).

3. **Structured vs. unstructured concurrency.** Structured concurrency (`TaskGroup`, child tasks) is used wherever work has a bounded, lexically scoped lifetime: backfill of multiple sessions, bidirectional sync push/pull, batch SwiftData writes. Unstructured `Task {}` is used only for long-running work whose lifetime is bound to user intent rather than a scope: the Spotify polling loop, the MusicKit observation loop, and periodic sync. Each unstructured `Task` handle is stored and explicitly cancelled when the session ends.

4. **@MainActor for UI.** `CaptureViewModel` and all other `@Observable` view models are `@MainActor`-isolated. They call into non-`@MainActor` actors via `await`, ensuring UI updates happen on the main thread without manual dispatching.

5. **ModelActor for SwiftData.** `PersistenceActor` uses the `@ModelActor` macro, which provides a dedicated `ModelContainer` and `ModelContext` confined to the actor's serial executor. This eliminates threading violations that would occur if `ModelContext` were shared across actors.

6. **Cancellation handling.** Every long-running loop checks `Task.isCancelled` or catches `CancellationError`. `AsyncStream.Continuation.finish()` is called on cancellation to ensure downstream `for await` loops terminate. `withTaskCancellationHandler` is used in `BGTaskScheduler` integration to bridge the system's expiration handler into Swift Concurrency's cancellation model.

7. **Error propagation.** Errors from capture services (network failures, auth expiration) are caught at the consumption-task level inside `CaptureEngine`. Transient errors trigger backoff retries within the service actor. Fatal errors (token revocation) bubble up via the `trackStream` finishing, which `CaptureEngine` detects and reports to the UI layer through `@Observable` state.

8. **Session limit enforcement.** `CaptureEngine` checks `activeSessions.count` before starting a new session. The `maxConcurrentSessions` value is injected at initialization (3 for free tier, uncapped for Pro) and can be updated when the user's subscription status changes.
