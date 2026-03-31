# ADR-006: Data Sync — Local-First SwiftData + Own API Server over CloudKit

**Status:** Accepted
**Date:** 2026-03-31
**Deciders:** Queueify Engineering

## Context

Queueify captures music listening sessions on-device and stores them in SwiftData. For free users, local storage is sufficient. But Pro users need:

1. **Server-side capture persistence** — always-on Spotify polling generates data on the server that must flow down to the device.
2. **Session backup** — sessions survive device loss or replacement.
3. **Collaborative sessions** — multiple users contribute to a shared session, requiring a shared data store.
4. **Cross-device access** — future Android app needs access to the same session data.

We need to decide how (and whether) to sync data between the device and a server.

### Constraints

- iOS 26, Swift 6+, SwiftData on-device, actors for sync engine
- The app must work fully offline — sync is additive, never required
- Track captures are append-only and immutable once written
- Session metadata (name, tags) is mutable
- Free tier: no sync, 30-day on-device retention
- Pro tier: bidirectional sync, unlimited retention
- Android is on the Phase 3 roadmap

## Decision

**Use a custom API server (Postgres) for sync, not CloudKit.** The app is local-first: all data is written to SwiftData immediately and the app functions fully offline. Pro users opt into bidirectional sync with our API server via REST, triggered by capture events and push notifications.

## Options Considered

### Option A: CloudKit + SwiftData Sync

Apple's built-in sync solution. SwiftData can be configured with a CloudKit-backed `ModelContainer` for automatic sync.

**Pros:**
- Zero server infrastructure to build or maintain
- Free (included with Apple Developer account, generous limits)
- Automatic conflict resolution
- Native SwiftData integration — minimal code

**Cons:**
- **Apple-only.** Android app (Phase 3) cannot access CloudKit data. We'd need to build a server sync layer later anyway and migrate all existing users.
- **No server-side writes.** Server-side Spotify polling generates captures on the server. CloudKit cannot receive writes from our backend — only from Apple devices. This is a fundamental blocker for the always-on capture Pro feature.
- **No shared containers across users.** CloudKit private databases are per-user. Collaborative sessions (multiple users contributing to one capture) require either a shared CloudKit container with complex permission management or a separate coordination mechanism.
- **Opaque conflict resolution.** CloudKit uses last-write-wins at the record level with no hooks for custom merge logic. We can't implement append-only semantics for track captures.
- **Schema constraints.** CloudKit has limits on indexes, record types, and relationships that may not align with our model as it evolves.

### Option B: Own API Server + Custom Sync (Chosen)

Build a REST API server backed by Postgres. The iOS app syncs via standard HTTP requests. Push notifications trigger sync pulls.

**Pros:**
- **Platform-agnostic.** Same API serves iOS now and Android later.
- **Server-side writes work naturally.** Always-on polling writes to Postgres; the device pulls those captures on next sync.
- **Collaborative sessions are straightforward.** Shared data lives in Postgres with standard relational access control.
- **Full control over conflict resolution.** We implement append-only track semantics and LWW for metadata exactly as needed.
- **Standard tooling.** Postgres, REST, push notifications — well-understood, easy to debug, easy to hire for.

**Cons:**
- Infrastructure cost (hosting, database, monitoring)
- More code to write and maintain (sync protocol, conflict resolution, retry logic)
- We own uptime and data durability

### Option C: Third-Party Sync (Firebase / Supabase Realtime)

Use a managed backend-as-a-service for sync.

**Pros:**
- Less infrastructure to manage than fully custom
- Real-time sync out of the box
- Cross-platform SDKs (Firebase supports iOS + Android)

**Cons:**
- **Vendor lock-in.** Firebase's data model (document store) doesn't map cleanly to our relational session/track model.
- **Cost at scale.** Firebase charges per read/write/storage — polling workers generating high write volume could be expensive.
- **Supabase Realtime is young.** Less battle-tested for mobile offline-first sync.
- **Still need server-side logic.** Always-on polling, collaborative session coordination, and playlist export still require a custom server — so we'd be running both a BaaS and a custom API.

## Rationale

Option B wins because:

1. **CloudKit is a non-starter** due to the server-side write requirement (always-on polling) and the Android roadmap. These aren't edge cases — they're core Pro features and a planned platform.
2. **We already need a server** for Pro's always-on Spotify polling, OAuth token custody, push notifications, and playlist export. Adding sync to that server is incremental, not a new system.
3. **Firebase/Supabase would be a second backend** alongside our custom server, adding complexity without eliminating the need for custom logic.
4. **Our sync requirements are simple.** Track captures are append-only (no update conflicts). Session metadata conflicts are rare and trivially resolved with LWW. We don't need a general-purpose sync framework — a focused protocol handles our use case with less code than integrating and configuring a third-party system.

## Consequences

### Positive

- **One backend for everything.** Sync, server-side polling, auth token custody, collaborative sessions, and playlist export all live in the same API server.
- **Android-ready from day one.** The API is platform-agnostic; the Android app will use the same sync protocol.
- **Predictable costs.** Postgres + a lightweight API server on Railway/Fly.io is cheap and scales linearly.
- **Full observability.** Standard server logs, metrics, and error tracking — no black-box sync.

### Negative

- **Infrastructure burden.** We own uptime, backups, and data durability. Mitigated by using managed Postgres (Supabase/RDS).
- **Sync code to maintain.** ~500-1000 lines of sync protocol code on client and server. Mitigated by the simplicity of the protocol (see below).
- **No automatic offline conflict resolution.** We handle it ourselves. Mitigated by the append-only nature of track captures making conflicts nearly nonexistent.

### Risks

| Risk | Likelihood | Mitigation |
|---|---|---|
| Sync bugs cause data loss | Low | Append-only tracks are immutable; soft-delete only; server-side backups |
| High sync traffic from active captures | Medium | Debounce and batch pushes (max every 30s); delta sync, not full sync |
| Push notification delivery unreliable | Medium | App also syncs on foreground; push is an optimization, not a requirement |
| Clock skew between device and server | Low | Server assigns canonical `synced_at` timestamps; device timestamps used only for LWW tiebreaking on metadata |

## Implementation Notes

### Sync Protocol

The sync protocol is pull-push over REST. Each syncable record carries a `sync_version` (monotonically increasing integer assigned by the server).

#### Push (device → server)

```
POST /api/v1/sync/push
Authorization: Bearer <session_token>
Content-Type: application/json

{
  "device_id": "uuid",
  "changes": [
    {
      "type": "session",
      "action": "upsert",
      "client_id": "uuid",
      "data": {
        "name": "Road Trip Day 2",
        "state": "completed",
        "started_at": "2026-03-31T14:00:00Z",
        "ended_at": "2026-03-31T18:30:00Z",
        "updated_at": "2026-03-31T18:30:00Z"
      }
    },
    {
      "type": "track",
      "action": "insert",
      "client_id": "uuid",
      "session_client_id": "uuid",
      "data": {
        "provider_track_id": "spotify:track:4PTG3Z6ehGkBFwjybzWkR8",
        "provider": "spotify",
        "title": "Never Gonna Give You Up",
        "artist": "Rick Astley",
        "album": "Whenever You Need Somebody",
        "album_art_url": "https://i.scdn.co/image/...",
        "duration_ms": 213000,
        "played_at": "2026-03-31T15:23:00Z"
      }
    }
  ]
}
```

**Server response:**

```json
{
  "accepted": 2,
  "rejected": 0,
  "server_version": 142,
  "errors": []
}
```

#### Pull (server → device)

```
GET /api/v1/sync/pull?since_version=130&limit=100
Authorization: Bearer <session_token>
```

**Response:**

```json
{
  "changes": [
    {
      "type": "track",
      "action": "insert",
      "server_id": "uuid",
      "client_id": null,
      "session_server_id": "uuid",
      "sync_version": 131,
      "source": "server_capture",
      "data": { ... }
    }
  ],
  "latest_version": 142,
  "has_more": true
}
```

The device stores `last_synced_version` and requests only changes since that version. Pagination via `limit` + `has_more` handles large deltas after extended offline periods.

#### Conflict Resolution

**Track captures (append-only):** No conflicts possible. Tracks are never updated or deleted. If the same track appears from both device and server capture (duplicate), deduplicate by `(session_id, provider_track_id, played_at)` — keep the first arrival, discard the duplicate.

**Session metadata (mutable):** Last-write-wins using `updated_at` timestamp.

```
Server has: { name: "Road Trip", updated_at: "2026-03-31T16:00:00Z" }
Device pushes: { name: "Epic Road Trip", updated_at: "2026-03-31T17:00:00Z" }
→ Device wins (later timestamp). Server updates and increments sync_version.

Server has: { name: "Road Trip", updated_at: "2026-03-31T18:00:00Z" }
Device pushes: { name: "Epic Road Trip", updated_at: "2026-03-31T17:00:00Z" }
→ Server wins (later timestamp). Server rejects the metadata change; device pulls server version on next sync.
```

**Session deletion:** Soft-delete only. A `deleted_at` timestamp propagates via sync. Tracks within a deleted session are retained server-side for 30 days before hard deletion.

### Postgres Schema

```sql
CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE provider_accounts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES users(id),
    provider        TEXT NOT NULL CHECK (provider IN ('spotify', 'apple_music')),
    provider_user_id TEXT NOT NULL,
    access_token_enc BYTEA,          -- AES-256-GCM encrypted
    refresh_token_enc BYTEA,         -- AES-256-GCM encrypted
    token_expires_at TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (user_id, provider)
);

CREATE TABLE capture_sessions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    client_id       UUID NOT NULL,   -- device-generated UUID
    user_id         UUID NOT NULL REFERENCES users(id),
    name            TEXT NOT NULL DEFAULT '',
    state           TEXT NOT NULL CHECK (state IN ('active', 'paused', 'completed')),
    provider        TEXT NOT NULL CHECK (provider IN ('spotify', 'apple_music')),
    started_at      TIMESTAMPTZ NOT NULL,
    ended_at        TIMESTAMPTZ,
    updated_at      TIMESTAMPTZ NOT NULL,
    deleted_at      TIMESTAMPTZ,
    sync_version    BIGINT NOT NULL DEFAULT 0,
    source          TEXT NOT NULL CHECK (source IN ('device', 'server')),
    UNIQUE (user_id, client_id)
);

CREATE INDEX idx_sessions_user_version ON capture_sessions (user_id, sync_version);
CREATE INDEX idx_sessions_user_state ON capture_sessions (user_id, state) WHERE deleted_at IS NULL;

CREATE TABLE captured_tracks (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    client_id       UUID NOT NULL,
    session_id      UUID NOT NULL REFERENCES capture_sessions(id),
    provider_track_id TEXT NOT NULL,
    provider        TEXT NOT NULL,
    title           TEXT NOT NULL,
    artist          TEXT NOT NULL,
    album           TEXT NOT NULL,
    album_art_url   TEXT,
    duration_ms     INTEGER NOT NULL,
    played_at       TIMESTAMPTZ NOT NULL,
    sync_version    BIGINT NOT NULL DEFAULT 0,
    source          TEXT NOT NULL CHECK (source IN ('device', 'server')),
    UNIQUE (session_id, provider_track_id, played_at)
);

CREATE INDEX idx_tracks_session ON captured_tracks (session_id, played_at);
CREATE INDEX idx_tracks_session_version ON captured_tracks (session_id, sync_version);

CREATE TABLE sync_cursors (
    user_id         UUID NOT NULL REFERENCES users(id),
    device_id       UUID NOT NULL,
    last_pulled_version BIGINT NOT NULL DEFAULT 0,
    last_pull_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (user_id, device_id)
);
```

### SyncEngine Actor (iOS)

```swift
actor SyncEngine {
    private let persistence: CaptureModelActor
    private let apiClient: APIClient
    private let pushRegistration: PushRegistration

    private var syncTask: Task<Void, Never>?
    private var lastSyncedVersion: Int64 = 0

    init(persistence: CaptureModelActor, apiClient: APIClient, pushRegistration: PushRegistration) {
        self.persistence = persistence
        self.apiClient = apiClient
        self.pushRegistration = pushRegistration
    }

    // MARK: - Sync Lifecycle

    func startPeriodicSync() {
        syncTask = Task {
            while !Task.isCancelled {
                await performSync()
                try? await Task.sleep(for: .seconds(60))
            }
        }
    }

    func stopSync() {
        syncTask?.cancel()
        syncTask = nil
    }

    /// Called when a push notification arrives indicating new server data
    func syncNow() async {
        await performSync()
    }

    // MARK: - Core Sync

    private func performSync() async {
        do {
            try await pushLocalChanges()
            try await pullRemoteChanges()
        } catch {
            // Log error, retry on next cycle — don't crash
        }
    }

    private func pushLocalChanges() async throws {
        let pendingChanges = await persistence.fetchPendingSync()
        guard !pendingChanges.isEmpty else { return }

        let request = SyncPushRequest(
            deviceId: DeviceIdentifier.current,
            changes: pendingChanges.map(\.toSyncPayload)
        )

        let response = try await apiClient.post("/api/v1/sync/push", body: request)

        // Mark successfully pushed records as synced
        await persistence.markSynced(
            ids: pendingChanges.map(\.clientId),
            serverVersion: response.serverVersion
        )
    }

    private func pullRemoteChanges() async throws {
        var hasMore = true

        while hasMore {
            let response = try await apiClient.get(
                "/api/v1/sync/pull",
                query: ["since_version": "\(lastSyncedVersion)", "limit": "100"]
            )

            for change in response.changes {
                switch change.type {
                case .session:
                    await persistence.mergeRemoteSession(change)
                case .track:
                    await persistence.mergeRemoteTrack(change)
                }
            }

            lastSyncedVersion = response.latestVersion
            hasMore = response.hasMore
        }
    }
}
```

### Sync Triggers

| Trigger | Action |
|---|---|
| Track captured on-device | Debounce 30s, then push batch |
| Capture session ended | Immediate push |
| App returns to foreground | Pull |
| Push notification received (silent) | Pull |
| Periodic timer (Pro) | Push + pull every 60s while app is active |
| User opens session list | Pull (if stale > 5 min) |

### Bandwidth & Battery Considerations

- **Debounced batching:** Track captures are batched and pushed at most every 30 seconds. A typical 3-minute song means ~1 track per batch — minimal payload.
- **Delta sync:** Only changes since `last_synced_version` are transferred. No full-state sync.
- **Payload size:** A single track record is ~500 bytes JSON. A 100-track session push is ~50KB — negligible.
- **Background sync:** Uses `BGProcessingTask` for extended sync after long offline periods. Not used for routine sync.
- **No WebSockets:** REST polling + push notifications keeps battery impact low. WebSockets maintain a persistent connection that drains battery — unnecessary for our sync cadence.
- **Conditional requests:** Pull requests include `If-None-Match` / `ETag` headers. Server returns `304 Not Modified` if nothing changed — no body transfer.

### Push Notification Integration

Server-side captures (always-on polling) trigger a silent push notification to the user's device:

```json
{
  "aps": {
    "content-available": 1
  },
  "sync_version": 143
}
```

The app receives this via `application(_:didReceiveRemoteNotification:fetchCompletionHandler:)`, calls `syncEngine.syncNow()`, and the pull fetches the new server-captured tracks. If the device is offline or the push is dropped, the next foreground sync catches up.

### Data Retention

| Tier | On-Device | Server |
|---|---|---|
| Free | 30 days (SwiftData cleanup via `CaptureModelActor`) | N/A |
| Pro | Unlimited (no cleanup) | Indefinite (soft-deleted sessions hard-deleted after 30 days) |

Free-to-Pro upgrade: existing on-device sessions (within 30-day window) are pushed to the server on first sync after subscription activation.
