# ADR-002: On-Device vs. Server-Side Capture Architecture

**Status:** Accepted
**Date:** 2026-03-31
**Deciders:** Queueify Engineering

## Context

Queueify captures music listening sessions from Spotify and Apple Music and saves them as playlists. A core product challenge is that iOS aggressively suspends and terminates background apps, which means any on-device capture strategy will have gaps when the user is not actively using the app. This ADR addresses the architectural question of where capture happens: on the user's device, on a server we control, or some combination of both.

The constraints that shape this decision are:

1. **Apple Music capture is device-only.** MusicKit's `ApplicationMusicPlayer` observation API (see ADR-001) is only available on-device via the native iOS framework. Apple does not expose a server-side API with per-play timestamps or real-time playback state. The Apple Music REST API's `recently-played` endpoint returns at most ~30 items with no `played_at` timestamps, making it unsuitable as a primary capture mechanism from a server.

2. **Spotify capture can work from a server.** Spotify's REST API (`GET /me/player/currently-playing` and `GET /me/player/recently-played`) can be called from any HTTP client with a valid OAuth access token. A server-side worker can poll these endpoints on behalf of a user without the app running at all.

3. **Background App Refresh is unreliable.** iOS controls when Background App Refresh (BAR) fires. Apple's documentation makes no guarantees about frequency or timing. In practice, BAR fires roughly every 15-30 minutes for apps the user opens frequently, but it can be delayed by hours or suppressed entirely by Low Power Mode, low battery, or the system's heuristic model of user behavior. BAR grants approximately 30 seconds of execution time per wake, which is enough for a backfill call but not for sustained polling.

4. **Business model split.** The product brief defines two tiers:
   - **Free tier:** On-device capture only. The app must be running (foreground) or recently backgrounded. Gaps are expected and backfilled best-effort from `recently-played` endpoints.
   - **Pro tier:** Server-side Spotify polling for always-on capture even when the app is suspended or terminated. On-device capture still runs in parallel when the app is active.

5. **Data persistence.** On-device data is stored in SwiftData (local SQLite). Pro users sync bidirectionally with an API server backed by Postgres. Free users do not sync.

6. **Target platform:** iOS 26+, Swift 6 with strict concurrency.

### Background App Refresh Gap Analysis

To quantify the capture gap that motivates server-side polling, consider a user who listens to music for 4 hours with the app backgrounded:

| Scenario | Tracks heard (est.) | Tracks captured via BAR backfill | Tracks lost |
|---|---|---|---|
| BAR fires every 15 min, Spotify `recently-played` returns 50 | ~114 (avg 210s/track) | Up to 50 (most recent at each wake) | ~64 |
| BAR fires every 30 min, Spotify `recently-played` returns 50 | ~114 | Up to 50 (with overlap between wakes) | ~64+ |
| BAR suppressed (Low Power Mode) | ~114 | 50 on next foreground | ~64 |
| BAR fires every 15 min, Apple Music returns 30 (no timestamps) | ~114 | Up to 30 (with dedup uncertainty) | ~84+ |

The fundamental issue: `recently-played` endpoints cap at 30-50 items, so any listening session longer than ~2 hours of continuous playback will lose tracks regardless of how often BAR fires. For power listeners on the Free tier, this is a known, accepted limitation. For Pro users willing to pay, server-side Spotify polling eliminates this gap entirely.

## Decision

**Hybrid architecture** -- on-device capture as the primary path for both Spotify and Apple Music, with optional server-side Spotify polling for Pro subscribers.

Specifically:

- **On-device capture (all users, both services):** Runs whenever the app is in the foreground or has an active background session. Uses the dual-strategy from ADR-001 (Spotify adaptive polling + Apple Music AsyncSequence observation). Backfills from `recently-played` on foreground and BAR wakes.
- **Server-side Spotify polling (Pro users only):** A per-user polling worker on our API server calls Spotify's `currently-playing` endpoint on behalf of the user. Captures are written to Postgres and synced to the device. This runs 24/7 regardless of app state.
- **Apple Music remains device-only (all tiers).** No server-side capture path exists or is planned for Apple Music. Pro users get the same Apple Music capture as Free users (on-device only), but their Spotify capture is gap-free.

## Options Considered

### Option A: On-device only

All capture happens on the iOS device for all users. No server infrastructure for capture. Rely on foreground observation, Background App Refresh backfill, and `recently-played` endpoint reconciliation to fill gaps.

**Advantages:**
- Zero server infrastructure cost for capture.
- No OAuth token storage on a server (reduced security surface).
- Simpler architecture -- one capture path, one data store.
- Works identically for Spotify and Apple Music.

**Disadvantages:**
- Guaranteed capture gaps during extended background/suspended periods. The `recently-played` cap (50 for Spotify, ~30 for Apple Music) means long listening sessions lose tracks permanently.
- BAR is unreliable -- iOS may suppress it for hours. Users in Low Power Mode get no background capture at all.
- No differentiation for a paid tier. The capture quality ceiling is the same for all users.
- User perception: "I listened for 3 hours and the app only caught the last 50 songs" is a frustrating experience that undermines trust in the core value proposition.

### Option B: Server-side only

All capture happens on the server. The iOS app is a thin client that displays and manages playlists. Spotify capture uses server-side polling. Apple Music capture uses... nothing viable, since MusicKit is device-only.

**Advantages:**
- Spotify capture is gap-free regardless of app state.
- Capture continues even if the user uninstalls and reinstalls the app.
- Centralized data store simplifies consistency.

**Disadvantages:**
- **Apple Music is completely unsupported.** This is a dealbreaker. MusicKit has no server-side equivalent that provides real-time playback state or per-play timestamps. Dropping Apple Music support eliminates a significant portion of the addressable market (Apple Music has ~90M+ subscribers).
- Requires all users to have a server account, even Free tier.
- Server must store and refresh OAuth tokens for every user, increasing security and compliance burden.
- Higher latency: server polls, writes to Postgres, then the app must fetch the data. On-device capture is immediate.
- Server cost scales linearly with active users -- every user needs a polling worker regardless of whether they are listening.

### Option C: Hybrid (on-device primary + server-side Pro)

On-device capture is the primary and universal path. Server-side Spotify polling is an additive layer for Pro users that fills the gaps left by iOS background execution limits. Apple Music capture remains device-only for all tiers. Data merges bidirectionally for Pro users.

**Advantages:**
- Apple Music is fully supported via on-device MusicKit observation.
- Free tier requires zero server infrastructure for capture.
- Pro tier provides genuine, measurable value: gap-free Spotify capture.
- On-device capture provides the lowest-latency, highest-fidelity path when the app is active -- even for Pro users, the on-device capture is better than server-side polling because it catches tracks the moment they start (vs. within a polling interval).
- Server cost scales only with Pro subscribers, not total users.
- Graceful degradation: if the server is down, Pro users still capture on-device. If the app is suspended, the server still captures Spotify.

**Disadvantages:**
- Two capture paths for Spotify Pro users means merge/dedup logic is required.
- Server infrastructure must be built and maintained for the polling workers.
- OAuth tokens must be stored server-side for Pro users (security surface).
- Apple Music Pro users get less value from their subscription (no server-side capture possible).

## Rationale

Option C (Hybrid) is the only architecture that satisfies all product and technical constraints:

1. **Apple Music requires on-device.** Options that abandon Apple Music (Option B) are not viable. Apple Music must be supported, and it can only be captured on-device. This alone forces on-device capture to be part of the architecture.

2. **On-device alone is insufficient for power users.** The BAR gap analysis shows that extended listening sessions will lose tracks. This is acceptable for a free tier but not for users willing to pay for reliability. Option A provides no upgrade path.

3. **Server-side cost is manageable when scoped to Pro.** Running a polling worker for every user (Option B) is expensive and wasteful. Running polling workers only for paying Pro subscribers aligns server cost with revenue. If 5% of users convert to Pro, we run 95% fewer polling workers than Option B.

4. **The hybrid model provides clear Pro tier value.** "Your Spotify sessions are captured 24/7, even when the app is closed" is a concrete, easy-to-understand benefit. This is harder to articulate with Option A ("we try harder in the background" is not compelling).

5. **Redundancy improves reliability.** For Pro Spotify users, the on-device and server-side capture paths are complementary. If the server misses a poll due to a deployment or outage, the on-device capture likely caught it (and vice versa). The union of both paths is strictly more complete than either alone.

## Consequences

### Positive

- **Gap-free Spotify capture for Pro users.** The server-side worker polls continuously, so no tracks are lost due to iOS background execution limits.
- **Apple Music fully supported.** On-device MusicKit observation works for all users at all tiers.
- **Free tier is zero server cost.** No capture infrastructure needed for users who do not pay.
- **Clear upgrade incentive.** The difference between Free and Pro is tangible and demonstrable (show users the gaps in their Free sessions vs. what Pro would have captured).
- **Resilient capture for Pro users.** Two independent capture paths (on-device + server) provide redundancy. A server outage does not mean lost captures when the app is active, and a backgrounded app does not mean lost Spotify captures.

### Negative

- **Merge complexity.** Pro Spotify users will have captures from two sources (on-device and server-side) that must be deduplicated and merged. The dedup logic from ADR-001 (trackID + timestamp tolerance) applies, but the tolerance window must account for clock skew between the device and server.
- **Asymmetric Pro value.** Users who only use Apple Music get less benefit from Pro (no server-side capture is possible). The Pro tier must include other features (e.g., sync, longer history, export) to justify the price for Apple Music-only users.
- **OAuth token custody.** Storing Spotify refresh tokens server-side increases the security surface. Tokens must be encrypted at rest, and the server must handle token refresh without user interaction.
- **Operational overhead.** The polling worker fleet must be monitored, scaled, and maintained. Worker crashes, Spotify API outages, and rate limiting all require alerting and graceful handling.

### Risks

- **Spotify API rate limiting at scale.** Each Pro user requires approximately 720 API calls/hour at a 5-second polling interval (12/min x 60). With 10,000 Pro users, that is 7.2 million calls/hour. Spotify's rate limits are per-app and not publicly documented with exact thresholds. Mitigation: use adaptive polling (slow down when idle), batch users across multiple Spotify app registrations if needed, and maintain a good relationship with Spotify's developer platform team.
- **Spotify ToS changes.** Spotify could restrict server-side polling or change their API terms to prohibit this use case. Mitigation: on-device capture is always the fallback. The app works (with gaps) without the server.
- **Clock skew in dedup.** The device and server have different clocks. A capture at device-time 14:00:03 and server-time 14:00:07 for the same track must be recognized as a duplicate. Mitigation: use a generous dedup tolerance window (60 seconds) keyed on (trackID, source: spotify, timestamp +/- 60s).
- **Cost overrun from idle polling.** If a Pro user stops using Spotify but remains subscribed, the worker still polls and receives "nothing playing" responses, consuming API quota and compute. Mitigation: implement an idle detection heuristic -- if a user has had no playback for 24 hours, reduce polling to once per 5 minutes; after 72 hours of inactivity, pause the worker entirely and resume on the next app sync or push notification.
- **Apple restricting MusicKit background execution.** Future iOS versions could further limit background execution. Mitigation: this affects all tiers equally and is mitigated by the existing backfill strategy. There is no alternative for Apple Music.

## Implementation Notes

### Server-Side Polling Worker Architecture

Each Pro user with a linked Spotify account gets a logical polling worker. The workers are implemented as lightweight async tasks managed by a worker pool, not as separate processes or containers.

```
                  +--------------------+
                  |   Worker Scheduler |
                  |   (Cron / Queue)   |
                  +---------+----------+
                            |
              +-------------+-------------+
              |             |             |
         +----v----+  +----v----+  +----v----+
         | Worker  |  | Worker  |  | Worker  |
         | User A  |  | User B  |  | User C  |
         +---------+  +---------+  +---------+
              |             |             |
              v             v             v
         Spotify API   Spotify API   Spotify API
         (OAuth per user)
              |             |             |
              v             v             v
         +--------------------------------------+
         |           Postgres                   |
         |  captured_tracks (server_source)     |
         +--------------------------------------+
```

**Worker lifecycle:**

1. The scheduler maintains a priority queue of (user_id, next_poll_at) tuples.
2. When `next_poll_at` arrives, a task from the pool picks up the user, refreshes their OAuth token if needed, calls `GET /me/player/currently-playing`, and processes the response.
3. The worker writes any new captures to the `captured_tracks` table with `source = 'server'`.
4. The worker computes the next poll interval (adaptive, same logic as on-device: 5s active, 30s idle, Retry-After on 429) and re-enqueues the user.
5. If the OAuth token is revoked (401 after refresh attempt), the worker marks the user as `polling_disabled` and sends a push notification prompting re-authentication.

**Estimated resource requirements:**

| Pro users | Polls/hour (active) | Polls/hour (idle mix) | Est. server cost |
|---|---|---|---|
| 1,000 | 720,000 | ~200,000 (est. 30% active) | 1 small VM or serverless tier |
| 10,000 | 7,200,000 | ~2,000,000 | 2-4 VMs or moderate serverless |
| 100,000 | 72,000,000 | ~20,000,000 | Dedicated worker fleet, multiple Spotify app keys |

Cost scales roughly linearly with Pro subscribers. At $5/month Pro pricing, the server cost per user is well within margin at the 1,000-10,000 range. Beyond 100,000 Pro users, Spotify API rate limits become the binding constraint rather than compute cost.

### Bidirectional Sync for Pro Users

Pro users have data in two places: SwiftData on-device and Postgres on the server. The sync protocol must handle:

1. **Server -> Device:** New server-captured tracks that the device has not seen. These are Spotify tracks captured while the app was suspended.
2. **Device -> Server:** New on-device captures (both Spotify and Apple Music) that the server has not seen. Apple Music captures only exist on-device and must be uploaded to Postgres for backup and cross-device access.
3. **Deduplication:** When both the device and server captured the same Spotify track, only one copy should survive. The dedup key is `(trackID, source, capturedAt +/- 60 seconds)`.

Sync runs on:
- App foreground (pull server captures, push device captures).
- Background App Refresh wake (same).
- After each server-side capture batch (push notification to wake the app, optional).

```swift
actor SyncCoordinator {
    private let localStore: CapturedTrackStore   // SwiftData
    private let apiClient: QueueifyAPIClient     // Server API

    /// Pull server captures that the device hasn't seen.
    func pullServerCaptures() async throws {
        let lastSyncTimestamp = await localStore.lastServerSyncTimestamp()
        let serverTracks = try await apiClient.getCaptures(since: lastSyncTimestamp)

        for track in serverTracks {
            let isDuplicate = await localStore.contains(
                trackID: track.trackID,
                capturedAt: track.capturedAt,
                toleranceSeconds: 60
            )
            if !isDuplicate {
                await localStore.insert(track, syncStatus: .synced)
            }
        }

        await localStore.updateServerSyncTimestamp(Date())
    }

    /// Push device captures that the server hasn't seen.
    func pushDeviceCaptures() async throws {
        let unsyncedTracks = await localStore.unsyncedTracks()
        if unsyncedTracks.isEmpty { return }

        try await apiClient.uploadCaptures(unsyncedTracks)

        for track in unsyncedTracks {
            await localStore.markSynced(track)
        }
    }

    /// Full bidirectional sync.
    func sync() async {
        do {
            try await pullServerCaptures()
            try await pushDeviceCaptures()
        } catch {
            // Log error, retry on next foreground
        }
    }
}
```

### On-Device Capture Data Flow (All Users)

```
+--------------------+     +--------------------+
| SpotifyCapture     |     | AppleMusicCapture  |
| Service (ADR-001)  |     | Service (ADR-001)  |
+--------+-----------+     +----------+---------+
         |                            |
         v                            v
+-------------------------------------------+
|       CaptureCoordinator (actor)          |
|  - Deduplicates across both sources       |
|  - Writes to SwiftData local store        |
|  - Groups tracks into sessions            |
+-------------------------------------------+
         |
         v
+-------------------------------------------+
|       SwiftData (local SQLite)            |
|  - CapturedTrack entities                 |
|  - Session entities                       |
|  - syncStatus: .unsynced / .synced        |
+-------------------------------------------+
         |
         | (Pro users only)
         v
+-------------------------------------------+
|       SyncCoordinator (actor)             |
|  - Pulls server captures                  |
|  - Pushes device captures                 |
|  - Deduplicates across device + server    |
+-------------------------------------------+
         |
         v
+-------------------------------------------+
|       Queueify API Server (Postgres)      |
+-------------------------------------------+
```

### Handling the Apple Music Pro Gap

Since Apple Music has no server-side capture path, Pro users who primarily use Apple Music may perceive less value. To address this:

1. **Be transparent.** The Pro tier description should clearly state that always-on capture applies to Spotify. Apple Music capture requires the app to be active or recently backgrounded.
2. **Maximize on-device Apple Music capture.** Use aggressive BAR scheduling, audio session tricks (if permitted by App Review), and backfill to minimize gaps.
3. **Bundle other Pro features.** Sync, extended history, playlist export to both services, and session analytics ensure Apple Music users still get value from Pro.
4. **Monitor for future Apple Music APIs.** If Apple ever exposes a server-callable playback history API with timestamps, add server-side Apple Music capture as a Pro feature immediately.
