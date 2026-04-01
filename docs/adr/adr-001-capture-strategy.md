# ADR-001: Capture Strategy — Spotify REST Polling vs. MusicKit AsyncSequence Observation

**Status:** Accepted
**Date:** 2026-03-31
**Deciders:** Queueify Engineering

## Context

Queueify captures music listening sessions from Spotify and Apple Music and saves them as playlists. To do this, we need a reliable mechanism to detect track changes in near-real-time while the user listens to music. The two services expose fundamentally different integration surfaces:

- **Spotify** offers no push-based notification or webhook for playback state. The only viable approach is polling the REST API (`GET /me/player/currently-playing`). The `recently-played` endpoint provides up to 50 tracks with `played_at` timestamps but is not real-time.
- **Apple Music** via MusicKit on iOS exposes `ApplicationMusicPlayer.shared.queue.currentEntry` as an observable property. Changes can be observed using Swift's `AsyncSequence` (`values` property on the publisher). The REST API `recently-played` endpoint is limited to ~30 items with no per-play timestamps, making it unsuitable as a primary capture mechanism.

Both services share a common problem: when the app is suspended or terminated, track changes are missed entirely. Background App Refresh provides a narrow window (~30 seconds) to catch up, but it is not guaranteed to fire and cannot sustain continuous polling. A backfill strategy using the respective `recently-played` endpoints is therefore required for both services on every app foregrounding event.

The app targets iOS 26+ with Swift 6 strict concurrency. All concurrency must use structured concurrency primitives: `async/await`, `AsyncSequence`, and `actor` isolation. Combine, GCD, and completion handlers are prohibited.

## Decision

We will use a **dual-strategy approach** with a unified capture protocol:

1. **Spotify:** Adaptive-interval REST polling via `Task.sleep(for:)` inside an actor-isolated loop.
2. **Apple Music:** `AsyncSequence` observation of `ApplicationMusicPlayer.shared.queue.currentEntry` inside an actor-isolated task.
3. **Both:** Backfill from `recently-played` on every app-foreground transition and on Background App Refresh wake.
4. **Both:** Actor-isolated deduplication using `(trackID, progress_ms, timestamp)` tuples.

## Options Considered

### Option A: Unified polling for both services

Poll both Spotify and Apple Music REST APIs on a fixed interval. This treats both services identically and simplifies the architecture.

**Rejected.** Apple Music's REST `recently-played` endpoint lacks per-play timestamps, making it impossible to accurately reconstruct a session from polling alone. MusicKit's on-device observation is strictly superior for Apple Music. Additionally, fixed-interval polling wastes API quota when the user is idle.

### Option B: Dual-strategy with adaptive polling (Spotify) and AsyncSequence observation (Apple Music)

Use the native strength of each platform: REST polling for Spotify (the only option) with an adaptive interval that speeds up during active playback and slows down when idle; `AsyncSequence` observation for Apple Music (zero-cost, event-driven, no API quota). Both share a common backfill path.

**Accepted.** This option maximizes capture fidelity while minimizing API calls and battery impact.

### Option C: Spotify Web Playback SDK + MusicKit observation

Use Spotify's Web Playback SDK to receive push events instead of polling.

**Rejected.** The Web Playback SDK requires an active browser context and is designed for web apps, not native iOS apps. It cannot run in the background on iOS. Spotify Connect's device-transfer model also complicates this: the SDK would need to become the active playback device, which disrupts the user's listening experience.

## Rationale

Option B is the only approach that respects the constraints of both platforms:

- Spotify has no push mechanism on iOS. Polling is the only option, but we can make it adaptive to reduce unnecessary network calls from ~720/hour (fixed 5s) to significantly fewer during idle periods.
- Apple Music's MusicKit provides a first-class observation API that is event-driven, battery-efficient, and does not count against any API quota. Using polling here would be strictly worse on every dimension.
- Both services lose visibility when the app is backgrounded. The `recently-played` backfill on foreground is essential regardless of the primary capture strategy, so it is shared infrastructure.
- Actor isolation gives us a clean concurrency model that satisfies Swift 6 strict concurrency without Combine or GCD.

## Consequences

### Positive

- **Optimal fidelity per platform.** Apple Music captures are event-driven with no delay. Spotify captures are bounded by the adaptive poll interval (5s during playback).
- **Battery and quota efficiency.** Apple Music observation is essentially free. Spotify polling backs off to 30s+ when idle.
- **Clean concurrency model.** Actors encapsulate all mutable state. No data races, no `@unchecked Sendable` hacks.
- **Shared backfill path.** Both services use the same backfill-on-foreground mechanism, reducing code duplication.
- **Testable.** The `CaptureService` protocol can be stubbed in tests with a simple `AsyncStream` of track events.

### Negative

- **Two code paths.** The Spotify and Apple Music capture implementations are structurally different (polling loop vs. async observation), which increases maintenance surface.
- **Spotify poll latency.** A track that plays for less than the poll interval (5s) could theoretically be missed. In practice, tracks under 5 seconds are rare and would be caught by the `recently-played` backfill.
- **Backfill has limits.** Spotify caps `recently-played` at 50 tracks; Apple Music at ~30. A user who listens to more than 50 tracks while the app is fully suspended will lose the oldest ones.

### Risks

- **Spotify rate limiting.** The Spotify API enforces rate limits (429 responses). The adaptive polling must respect `Retry-After` headers and back off exponentially. If rate limits tighten in the future, capture fidelity could degrade.
- **MusicKit API changes.** Apple could change the observation API surface in future iOS releases. The actor boundary insulates the rest of the app, but the `AppleMusicCaptureService` itself may need updates.
- **Background App Refresh unreliability.** iOS makes no guarantees about BAR timing or frequency. Users who keep the app backgrounded for extended periods will rely entirely on the `recently-played` backfill, which is lossy.

## Implementation Notes

### Shared Protocol

All capture services conform to a common protocol that emits a stream of `CapturedTrack` values:

```swift
struct CapturedTrack: Sendable, Equatable {
    let trackID: String          // Spotify track URI or Apple Music catalog ID
    let title: String
    let artist: String
    let albumName: String
    let durationMs: Int
    let capturedAt: Date         // Wall-clock time we observed this track
    let source: MusicService     // .spotify | .appleMusic
}

enum MusicService: String, Sendable {
    case spotify
    case appleMusic
}

protocol CaptureService: Actor {
    func startCapturing() -> AsyncStream<CapturedTrack>
    func stopCapturing()
    func backfillRecentlyPlayed() async throws -> [CapturedTrack]
}
```

### Spotify Capture Service (Adaptive Polling)

```swift
actor SpotifyCaptureService: CaptureService {
    private var isCapturing = false
    private var lastSeenTrackID: String?
    private var lastSeenProgressMs: Int?
    private var lastSeenTimestamp: Date?
    private var captureTask: Task<Void, Never>?

    private let apiClient: SpotifyAPIClient

    // Adaptive interval boundaries
    private let activeInterval: Duration = .seconds(5)
    private let idleInterval: Duration = .seconds(30)
    private let cooldownAfterIdle: Duration = .seconds(10)
    private var currentInterval: Duration = .seconds(5)

    init(apiClient: SpotifyAPIClient) {
        self.apiClient = apiClient
    }

    func startCapturing() -> AsyncStream<CapturedTrack> {
        let (stream, continuation) = AsyncStream<CapturedTrack>.makeStream()
        isCapturing = true

        captureTask = Task { [weak self] in
            guard let self else { return }
            while !Task.isCancelled {
                let interval = await self.poll(continuation: continuation)
                try? await Task.sleep(for: interval)
            }
            continuation.finish()
        }

        return stream
    }

    func stopCapturing() {
        isCapturing = false
        captureTask?.cancel()
        captureTask = nil
    }

    /// Performs one poll cycle. Returns the interval to wait before the next poll.
    private func poll(continuation: AsyncStream<CapturedTrack>.Continuation) async -> Duration {
        do {
            let response = try await apiClient.getCurrentlyPlaying()

            guard let playback = response, playback.isPlaying else {
                // Nothing playing — back off to idle interval
                lastSeenTrackID = nil
                lastSeenProgressMs = nil
                currentInterval = idleInterval
                return currentInterval
            }

            let trackID = playback.item.uri
            let progressMs = playback.progressMs
            let now = Date()

            if isDuplicate(trackID: trackID, progressMs: progressMs, timestamp: now) {
                // Same track, progress advancing normally — not a new capture event
                currentInterval = activeInterval
                return currentInterval
            }

            // New track detected (or replay detected)
            let track = CapturedTrack(
                trackID: trackID,
                title: playback.item.name,
                artist: playback.item.artists.first?.name ?? "Unknown",
                albumName: playback.item.album.name,
                durationMs: playback.item.durationMs,
                capturedAt: now,
                source: .spotify
            )
            continuation.yield(track)

            lastSeenTrackID = trackID
            lastSeenProgressMs = progressMs
            lastSeenTimestamp = now
            currentInterval = activeInterval
            return currentInterval

        } catch let error as SpotifyAPIError where error.isRateLimited {
            // Respect Retry-After header
            let retryAfter = error.retryAfterSeconds ?? 30
            return .seconds(retryAfter)
        } catch {
            // Network error — use cooldown interval, don't lose state
            currentInterval = cooldownAfterIdle
            return currentInterval
        }
    }

    /// Deduplication logic using track ID + progress_ms + timestamp.
    ///
    /// A poll is considered a duplicate (i.e., continued playback of the same track)
    /// when ALL of the following are true:
    ///   1. The track ID matches the last-seen track ID.
    ///   2. progress_ms has advanced by roughly the elapsed wall-clock time
    ///      (within a tolerance window to account for seek jitter and poll timing).
    ///
    /// If the track ID matches but progress_ms has RESET (gone back to near zero
    /// or jumped backward significantly), we treat it as a REPLAY — a new listen
    /// event for the same track.
    private func isDuplicate(trackID: String, progressMs: Int, timestamp: Date) -> Bool {
        guard
            let lastID = lastSeenTrackID,
            let lastProgress = lastSeenProgressMs,
            let lastTimestamp = lastSeenTimestamp,
            lastID == trackID
        else {
            // Different track or first poll — not a duplicate
            return false
        }

        let elapsedMs = Int(timestamp.timeIntervalSince(lastTimestamp) * 1000)
        let expectedProgress = lastProgress + elapsedMs
        let tolerance = 3000 // 3 seconds tolerance for network/poll jitter

        // If progress is roughly where we expect, it's continued playback (duplicate)
        if abs(progressMs - expectedProgress) < tolerance {
            return true
        }

        // If progress jumped backward significantly, it's a replay (not duplicate)
        if progressMs < lastProgress - tolerance {
            return false
        }

        // Edge case: progress jumped forward (seek). Treat as continued playback.
        return true
    }

    func backfillRecentlyPlayed() async throws -> [CapturedTrack] {
        let response = try await apiClient.getRecentlyPlayed(limit: 50)
        return response.items.map { item in
            CapturedTrack(
                trackID: item.track.uri,
                title: item.track.name,
                artist: item.track.artists.first?.name ?? "Unknown",
                albumName: item.track.album.name,
                durationMs: item.track.durationMs,
                capturedAt: item.playedAt,  // Spotify provides played_at timestamps
                source: .spotify
            )
        }
    }
}
```

### Apple Music Capture Service (AsyncSequence Observation)

```swift
actor AppleMusicCaptureService: CaptureService {
    private var isCapturing = false
    private var lastSeenTrackID: String?
    private var observationTask: Task<Void, Never>?

    func startCapturing() -> AsyncStream<CapturedTrack> {
        let (stream, continuation) = AsyncStream<CapturedTrack>.makeStream()
        isCapturing = true

        observationTask = Task { [weak self] in
            let player = ApplicationMusicPlayer.shared

            // Observe currentEntry changes via withObservationTracking polling loop.
            // This avoids Combine's objectWillChange — instead we use the Observation
            // framework to detect when currentEntry changes, yielding an AsyncStream.
            let entryChanges = AsyncStream<ApplicationMusicPlayer.Queue.Entry?> { continuation in
                @Sendable func observe() {
                    withObservationTracking {
                        _ = player.queue.currentEntry
                    } onChange: {
                        continuation.yield(player.queue.currentEntry)
                        observe()
                    }
                }
                observe()
                continuation.onTermination = { _ in }
            }

            for await entry in entryChanges {
                guard !Task.isCancelled else { break }
                guard let self else { break }

                guard let entry, case let .song(song) = entry.item else {
                    continue
                }

                let trackID = song.id.rawValue
                let now = Date()

                // Allow replays: skip only if same track AND less than track duration elapsed
                let isDuplicate: Bool
                if await self.lastSeenTrackID == trackID {
                    let elapsed = now.timeIntervalSince(await self.lastSeenTime)
                    let duration = song.duration ?? 180 // default 3 min if unknown
                    isDuplicate = elapsed < duration * 0.8 // within 80% of track length = still playing
                } else {
                    isDuplicate = false
                }
                guard !isDuplicate else { continue }

                let track = CapturedTrack(
                    trackID: trackID,
                    title: song.title,
                    artist: song.artistName,
                    albumName: song.albumTitle ?? "Unknown",
                    durationMs: Int((song.duration ?? 0) * 1000),
                    capturedAt: Date(),
                    source: .appleMusic
                )
                continuation.yield(track)
                await self.updateLastSeen(trackID: trackID, at: now)
            }

            continuation.finish()
        }

        return stream
    }

    private func updateLastSeen(trackID: String) {
        lastSeenTrackID = trackID
    }

    func stopCapturing() {
        isCapturing = false
        observationTask?.cancel()
        observationTask = nil
    }

    func backfillRecentlyPlayed() async throws -> [CapturedTrack] {
        // MusicKit recently played — no per-play timestamps available,
        // so we use Date() as a fallback and rely on ordering.
        var request = MusicRecentlyPlayedRequest()
        request.limit = 30
        let response = try await request.response()

        let now = Date()
        return response.items.enumerated().map { index, item in
            CapturedTrack(
                trackID: item.id.rawValue,
                title: item.title,
                artist: item.artistName,
                albumName: item.albumTitle ?? "Unknown",
                durationMs: Int((item.duration ?? 0) * 1000),
                // Approximate ordering: most recent first, offset by estimated duration
                capturedAt: now.addingTimeInterval(-Double(index) * 210),
                source: .appleMusic
            )
        }
    }
}
```

### Backfill Coordinator

The backfill coordinator runs on every `scenePhase` transition to `.active` and reconciles recently-played tracks against already-captured tracks in the local store:

```swift
actor BackfillCoordinator {
    private let store: CapturedTrackStore

    init(store: CapturedTrackStore) {
        self.store = store
    }

    /// Called on app foreground. Fetches recently-played from the given
    /// capture service and inserts any tracks not already in the store.
    func backfill(from service: any CaptureService) async {
        do {
            let recentTracks = try await service.backfillRecentlyPlayed()

            for track in recentTracks {
                let isDuplicate = await store.contains(
                    trackID: track.trackID,
                    capturedAt: track.capturedAt,
                    toleranceSeconds: 60
                )
                if !isDuplicate {
                    await store.insert(track)
                }
            }
        } catch {
            // Log but don't crash — backfill is best-effort
        }
    }
}
```

### Background App Refresh Registration

```swift
func registerBackgroundRefresh() {
    BGTaskScheduler.shared.register(
        forTaskWithIdentifier: "com.queueify.backfill",
        using: nil
    ) { task in
        Task {
            let coordinator = BackfillCoordinator(store: sharedStore)

            if spotifySession.isAuthenticated {
                await coordinator.backfill(from: spotifyCaptureService)
            }
            if appleMusicSession.isAuthorized {
                await coordinator.backfill(from: appleMusicCaptureService)
            }

            task.setTaskCompleted(success: true)
        }

        // Schedule the next refresh
        scheduleBackgroundRefresh()
    }
}

func scheduleBackgroundRefresh() {
    let request = BGAppRefreshTaskRequest(
        identifier: "com.queueify.backfill"
    )
    request.earliestBeginDate = Date(timeIntervalSinceNow: 15 * 60)
    try? BGTaskScheduler.shared.submit(request)
}
```

### Adaptive Polling Interval Summary

| State | Interval | Rationale |
|---|---|---|
| Music actively playing | 5 seconds | Fast enough to catch most tracks (median pop track ~210s) |
| Nothing playing / paused | 30 seconds | Reduce API calls; user may resume at any time |
| After network error | 10 seconds | Brief cooldown before retry |
| Rate limited (429) | `Retry-After` header value | Mandatory compliance with Spotify API contract |

### Deduplication Decision Tree

```
Is track ID different from last seen?
  YES -> Emit as new capture
  NO  -> Check progress_ms:
           Did progress_ms reset to near zero (or jump backward)?
             YES -> Emit as replay (user hit repeat)
             NO  -> Is progress_ms advancing roughly in line with wall-clock?
                      YES -> Suppress (continued playback, duplicate)
                      NO  -> Progress jumped forward (seek) — suppress (still same listen)
```
