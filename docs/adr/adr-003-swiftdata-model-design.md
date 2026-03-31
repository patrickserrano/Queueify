# ADR-003: SwiftData Model Design for Sessions, Tracks, and Provider Accounts

**Status:** Accepted
**Date:** 2026-03-31
**Deciders:** Queueify Engineering

## Context

Queueify captures music listening sessions from Spotify and Apple Music and saves them as playlists. The persistence layer must store three core concepts:

1. **Capture sessions** — a time-bounded recording of what the user listened to. Sessions have a lifecycle (`active` -> `paused` -> `completed`) and are associated with a single source provider. Free-tier users are limited to 3 concurrent active sessions and 30 days of history. Pro-tier users get unlimited history and collaborative sessions.
2. **Captured tracks** — individual track records appended to a session during capture. Tracks are immutable once written; they are never edited or reordered after capture. Each track carries provider-specific identifiers, metadata (title, artist, album, album art URL, duration), and a `played_at` timestamp.
3. **Provider accounts** — linked Spotify and/or Apple Music accounts. A user may connect one or both providers. Each account stores authentication tokens, expiration, and display metadata.

Additional requirements:

- The app targets **iOS 26+ with Swift 6 strict concurrency**. All persistence must use **SwiftData** with the `@Model` macro. Core Data and `NSManagedObject` are prohibited.
- Background writes (track appends during capture) must happen off the main actor using `ModelActor`.
- Pro-tier users sync sessions to the server. Each syncable record needs a sync state (`pending`, `synced`, `conflict`) to support optimistic local-first writes with eventual server reconciliation.
- Common query patterns include: all sessions sorted by start time, tracks within a session sorted by `played_at`, sessions older than 30 days (for retention cleanup), and sessions/tracks in a given sync state.
- Indexes are required for fields used in predicates and sort descriptors to keep list views performant as history grows.

## Decision

We will define four `@Model` classes — `CaptureSession`, `CapturedTrack`, `ProviderAccount`, and `SyncMetadata` — with explicit SwiftData relationships, indexes, and `#Predicate`-based queries. All background writes will go through a dedicated `ModelActor` to avoid blocking the main thread during capture. Retention cleanup will use a `#Predicate` over `endTime` to batch-delete expired sessions for free-tier users.

### Complete Model Definitions

```swift
import Foundation
import SwiftData

// MARK: - Enums

enum SessionState: String, Codable, Sendable {
    case active
    case paused
    case completed
}

enum MusicProvider: String, Codable, Sendable {
    case spotify
    case appleMusic = "apple_music"
}

enum SyncState: String, Codable, Sendable {
    case pending
    case synced
    case conflict
}

enum SubscriptionTier: String, Codable, Sendable {
    case free
    case pro
}

// MARK: - SyncMetadata

@Model
final class SyncMetadata: Sendable {
    var syncState: SyncState
    var lastSyncedAt: Date?
    var serverRevision: Int
    var localRevision: Int
    var conflictPayload: Data?

    init(
        syncState: SyncState = .pending,
        lastSyncedAt: Date? = nil,
        serverRevision: Int = 0,
        localRevision: Int = 1,
        conflictPayload: Data? = nil
    ) {
        self.syncState = syncState
        self.lastSyncedAt = lastSyncedAt
        self.serverRevision = serverRevision
        self.localRevision = localRevision
        self.conflictPayload = conflictPayload
    }
}

// MARK: - ProviderAccount

@Model
final class ProviderAccount: Sendable {
    #Unique<ProviderAccount>([\.provider, \.providerUserID])
    #Index<ProviderAccount>([\.provider])

    @Attribute(.unique)
    var id: UUID

    var provider: MusicProvider
    var providerUserID: String
    var displayName: String
    var email: String?
    var profileImageURL: URL?

    var accessToken: String
    var refreshToken: String?
    var tokenExpiresAt: Date

    var connectedAt: Date
    var lastUsedAt: Date

    @Relationship(inverse: \CaptureSession.providerAccount)
    var sessions: [CaptureSession] = []

    init(
        id: UUID = UUID(),
        provider: MusicProvider,
        providerUserID: String,
        displayName: String,
        email: String? = nil,
        profileImageURL: URL? = nil,
        accessToken: String,
        refreshToken: String? = nil,
        tokenExpiresAt: Date,
        connectedAt: Date = .now,
        lastUsedAt: Date = .now
    ) {
        self.id = id
        self.provider = provider
        self.providerUserID = providerUserID
        self.displayName = displayName
        self.email = email
        self.profileImageURL = profileImageURL
        self.accessToken = accessToken
        self.refreshToken = refreshToken
        self.tokenExpiresAt = tokenExpiresAt
        self.connectedAt = connectedAt
        self.lastUsedAt = lastUsedAt
    }
}

// MARK: - CaptureSession

@Model
final class CaptureSession: Sendable {
    #Index<CaptureSession>([\.startTime], [\.state], [\.sourceProvider])

    @Attribute(.unique)
    var id: UUID

    var name: String
    var tags: [String]

    var startTime: Date
    var endTime: Date?

    var state: SessionState
    var sourceProvider: MusicProvider

    @Relationship(deleteRule: .cascade)
    var tracks: [CapturedTrack] = []

    @Relationship
    var providerAccount: ProviderAccount?

    @Relationship(deleteRule: .cascade)
    var syncMetadata: SyncMetadata?

    // Collaborative session support (Pro tier)
    var isCollaborative: Bool
    var collaboratorIDs: [String]

    init(
        id: UUID = UUID(),
        name: String,
        tags: [String] = [],
        startTime: Date = .now,
        endTime: Date? = nil,
        state: SessionState = .active,
        sourceProvider: MusicProvider,
        providerAccount: ProviderAccount? = nil,
        syncMetadata: SyncMetadata? = nil,
        isCollaborative: Bool = false,
        collaboratorIDs: [String] = []
    ) {
        self.id = id
        self.name = name
        self.tags = tags
        self.startTime = startTime
        self.endTime = endTime
        self.state = state
        self.sourceProvider = sourceProvider
        self.providerAccount = providerAccount
        self.syncMetadata = syncMetadata
        self.isCollaborative = isCollaborative
        self.collaboratorIDs = collaboratorIDs
    }
}

// MARK: - CapturedTrack

@Model
final class CapturedTrack: Sendable {
    #Index<CapturedTrack>([\.playedAt], [\.providerTrackID])

    @Attribute(.unique)
    var id: UUID

    /// Provider-specific track identifier (e.g., Spotify URI, Apple Music catalog ID).
    var providerTrackID: String
    var source: MusicProvider

    var title: String
    var artist: String
    var album: String
    var albumArtURL: URL?
    var durationSeconds: TimeInterval

    var playedAt: Date

    @Relationship(inverse: \CaptureSession.tracks)
    var session: CaptureSession?

    @Relationship(deleteRule: .cascade)
    var syncMetadata: SyncMetadata?

    init(
        id: UUID = UUID(),
        providerTrackID: String,
        source: MusicProvider,
        title: String,
        artist: String,
        album: String,
        albumArtURL: URL? = nil,
        durationSeconds: TimeInterval,
        playedAt: Date,
        session: CaptureSession? = nil,
        syncMetadata: SyncMetadata? = nil
    ) {
        self.id = id
        self.providerTrackID = providerTrackID
        self.source = source
        self.title = title
        self.artist = artist
        self.album = album
        self.albumArtURL = albumArtURL
        self.durationSeconds = durationSeconds
        self.playedAt = playedAt
        self.session = session
        self.syncMetadata = syncMetadata
    }
}
```

### ModelContainer Configuration

```swift
import SwiftData

enum QueueifyModelContainer {
    static let schema = Schema([
        CaptureSession.self,
        CapturedTrack.self,
        ProviderAccount.self,
        SyncMetadata.self,
    ])

    static func create(inMemory: Bool = false) throws -> ModelContainer {
        let configuration = ModelConfiguration(
            "Queueify",
            schema: schema,
            isStoredInMemoryOnly: inMemory,
            allowsSave: true
        )
        return try ModelContainer(
            for: schema,
            configurations: [configuration]
        )
    }
}
```

### ModelActor for Background Writes

```swift
import SwiftData

@ModelActor
actor CaptureModelActor {

    // MARK: - Track Append (hot path during capture)

    func appendTrack(
        to sessionID: UUID,
        providerTrackID: String,
        source: MusicProvider,
        title: String,
        artist: String,
        album: String,
        albumArtURL: URL?,
        durationSeconds: TimeInterval,
        playedAt: Date
    ) throws {
        let predicate = #Predicate<CaptureSession> { $0.id == sessionID }
        let descriptor = FetchDescriptor(predicate: predicate)

        guard let session = try modelContext.fetch(descriptor).first else {
            throw QueueifyPersistenceError.sessionNotFound(sessionID)
        }

        let track = CapturedTrack(
            providerTrackID: providerTrackID,
            source: source,
            title: title,
            artist: artist,
            album: album,
            albumArtURL: albumArtURL,
            durationSeconds: durationSeconds,
            playedAt: playedAt,
            session: session,
            syncMetadata: SyncMetadata()
        )

        modelContext.insert(track)
        session.tracks.append(track)
        try modelContext.save()
    }

    // MARK: - Session Lifecycle

    func createSession(
        name: String,
        sourceProvider: MusicProvider,
        providerAccountID: UUID,
        tier: SubscriptionTier
    ) throws -> UUID {
        // Enforce free-tier active session limit.
        if tier == .free {
            let activeState = SessionState.active
            let predicate = #Predicate<CaptureSession> { $0.state == activeState }
            let descriptor = FetchDescriptor(predicate: predicate)
            let activeCount = try modelContext.fetchCount(descriptor)

            guard activeCount < 3 else {
                throw QueueifyPersistenceError.activeSessionLimitReached
            }
        }

        // Resolve the provider account.
        let accountPredicate = #Predicate<ProviderAccount> {
            $0.id == providerAccountID
        }
        let accountDescriptor = FetchDescriptor(predicate: accountPredicate)
        let account = try modelContext.fetch(accountDescriptor).first

        let session = CaptureSession(
            name: name,
            sourceProvider: sourceProvider,
            providerAccount: account,
            syncMetadata: SyncMetadata()
        )

        modelContext.insert(session)
        try modelContext.save()
        return session.id
    }

    func transitionSession(_ sessionID: UUID, to newState: SessionState) throws {
        let predicate = #Predicate<CaptureSession> { $0.id == sessionID }
        let descriptor = FetchDescriptor(predicate: predicate)

        guard let session = try modelContext.fetch(descriptor).first else {
            throw QueueifyPersistenceError.sessionNotFound(sessionID)
        }

        session.state = newState
        if newState == .completed {
            session.endTime = .now
        }

        session.syncMetadata?.syncState = .pending
        session.syncMetadata?.localRevision += 1
        try modelContext.save()
    }

    // MARK: - Retention Cleanup (Free Tier)

    func deleteExpiredSessions(retentionDays: Int = 30) throws -> Int {
        let cutoff = Calendar.current.date(
            byAdding: .day, value: -retentionDays, to: .now
        )!
        let completedState = SessionState.completed

        let predicate = #Predicate<CaptureSession> {
            $0.state == completedState && $0.endTime != nil && $0.endTime! < cutoff
        }

        let descriptor = FetchDescriptor(predicate: predicate)
        let expired = try modelContext.fetch(descriptor)
        let count = expired.count

        for session in expired {
            modelContext.delete(session) // cascade deletes tracks + syncMetadata
        }

        try modelContext.save()
        return count
    }

    // MARK: - Sync State Management (Pro Tier)

    func fetchPendingSyncSessions() throws -> [CaptureSession] {
        let pendingState = SyncState.pending
        let predicate = #Predicate<CaptureSession> {
            $0.syncMetadata?.syncState == pendingState
        }
        let descriptor = FetchDescriptor(predicate: predicate)
        return try modelContext.fetch(descriptor)
    }

    func markSessionSynced(
        _ sessionID: UUID,
        serverRevision: Int
    ) throws {
        let predicate = #Predicate<CaptureSession> { $0.id == sessionID }
        let descriptor = FetchDescriptor(predicate: predicate)

        guard let session = try modelContext.fetch(descriptor).first else {
            throw QueueifyPersistenceError.sessionNotFound(sessionID)
        }

        session.syncMetadata?.syncState = .synced
        session.syncMetadata?.serverRevision = serverRevision
        session.syncMetadata?.lastSyncedAt = .now
        try modelContext.save()
    }

    func markSessionConflict(
        _ sessionID: UUID,
        conflictPayload: Data
    ) throws {
        let predicate = #Predicate<CaptureSession> { $0.id == sessionID }
        let descriptor = FetchDescriptor(predicate: predicate)

        guard let session = try modelContext.fetch(descriptor).first else {
            throw QueueifyPersistenceError.sessionNotFound(sessionID)
        }

        session.syncMetadata?.syncState = .conflict
        session.syncMetadata?.conflictPayload = conflictPayload
        try modelContext.save()
    }
}

// MARK: - Errors

enum QueueifyPersistenceError: Error, Sendable {
    case sessionNotFound(UUID)
    case activeSessionLimitReached
    case providerAccountNotFound(UUID)
}
```

### Common Query Predicates and Descriptors

```swift
import SwiftData

enum SessionQueries {

    /// All sessions, most recent first.
    static func allByStartTime() -> FetchDescriptor<CaptureSession> {
        var descriptor = FetchDescriptor<CaptureSession>(
            sortBy: [SortDescriptor(\.startTime, order: .reverse)]
        )
        descriptor.relationshipKeyPathsForPrefetching = [\.tracks]
        return descriptor
    }

    /// Only active sessions (for the "Now Capturing" view).
    static func activeSessions() -> FetchDescriptor<CaptureSession> {
        let activeState = SessionState.active
        let predicate = #Predicate<CaptureSession> { $0.state == activeState }
        return FetchDescriptor(
            predicate: predicate,
            sortBy: [SortDescriptor(\.startTime, order: .reverse)]
        )
    }

    /// Sessions for a specific provider.
    static func sessions(
        for provider: MusicProvider
    ) -> FetchDescriptor<CaptureSession> {
        let predicate = #Predicate<CaptureSession> {
            $0.sourceProvider == provider
        }
        return FetchDescriptor(
            predicate: predicate,
            sortBy: [SortDescriptor(\.startTime, order: .reverse)]
        )
    }

    /// Sessions completed before a cutoff date (retention cleanup candidates).
    static func expiredSessions(
        before cutoff: Date
    ) -> FetchDescriptor<CaptureSession> {
        let completedState = SessionState.completed
        let predicate = #Predicate<CaptureSession> {
            $0.state == completedState
                && $0.endTime != nil
                && $0.endTime! < cutoff
        }
        return FetchDescriptor(predicate: predicate)
    }

    /// Tracks within a session, ordered by play time.
    static func tracks(
        inSession sessionID: UUID
    ) -> FetchDescriptor<CapturedTrack> {
        let predicate = #Predicate<CapturedTrack> {
            $0.session?.id == sessionID
        }
        return FetchDescriptor(
            predicate: predicate,
            sortBy: [SortDescriptor(\.playedAt, order: .forward)]
        )
    }

    /// Tracks needing sync (Pro tier).
    static func pendingSyncTracks() -> FetchDescriptor<CapturedTrack> {
        let pendingState = SyncState.pending
        let predicate = #Predicate<CapturedTrack> {
            $0.syncMetadata?.syncState == pendingState
        }
        return FetchDescriptor(predicate: predicate)
    }
}
```

### SwiftUI Integration (Main Actor Reads)

```swift
import SwiftUI
import SwiftData

struct SessionListView: View {
    @Query(sort: \CaptureSession.startTime, order: .reverse)
    private var sessions: [CaptureSession]

    var body: some View {
        List(sessions) { session in
            SessionRow(session: session)
        }
    }
}

struct ActiveSessionsView: View {
    static let activeState = SessionState.active

    @Query(
        filter: #Predicate<CaptureSession> { $0.state == activeState },
        sort: \CaptureSession.startTime,
        order: .reverse
    )
    private var activeSessions: [CaptureSession]

    var body: some View {
        List(activeSessions) { session in
            ActiveSessionRow(session: session)
        }
    }
}
```

## Options Considered

### Option A: Single Flat Model (Session with Embedded Track Data)

Store tracks as a `Codable` array directly on the session model instead of as separate `@Model` entities.

- **Pro:** Simpler schema, single fetch loads everything, no relationship management.
- **Con:** Every track append rewrites the entire array. No way to index individual track fields. No incremental sync — the entire session blob must be uploaded on every change. SwiftData change tracking operates at the property level, so every append marks the whole array dirty.

### Option B: Core Data with Manual Migration

Use Core Data directly with `NSManagedObject` subclasses and a manual migration plan.

- **Con:** Violates the project's iOS 26+/Swift 6 constraints. Core Data's `NSManagedObject` is not `Sendable` and fights structured concurrency. SwiftData's `@Model` macro, `#Predicate`, and `ModelActor` are purpose-built for this stack.

### Option C: SwiftData with Normalized Models and ModelActor (Chosen)

Four distinct `@Model` classes with explicit relationships, `ModelActor` for background writes, and `#Predicate`-based queries.

- **Pro:** Each track is an independent row — appends are O(1), individual tracks are indexable, and sync can operate at track granularity. `ModelActor` keeps capture writes off the main thread. Cascade delete rules ensure session deletion cleans up tracks and sync metadata automatically. `#Index` on `startTime`, `playedAt`, `state`, and `providerTrackID` covers all hot query paths.
- **Con:** More classes to maintain. Relationships add complexity. Requires care to avoid N+1 fetches when loading session lists with track counts.

## Rationale

**Option C is chosen** for the following reasons:

1. **Append-only track writes are the hottest write path.** During an active capture session, tracks are appended every few seconds. Normalized `CapturedTrack` rows mean each insert is a single-row write with no contention on the parent session. Embedding tracks as a `Codable` array (Option A) would rewrite the entire blob on every append, degrading write performance linearly with session length.

2. **Granular sync for Pro tier.** With tracks as independent entities carrying their own `SyncMetadata`, the sync engine can upload individual tracks as they are captured rather than waiting for the session to complete. This enables real-time collaborative sessions where multiple users see tracks appear live.

3. **ModelActor aligns with Swift 6 strict concurrency.** The `@ModelActor` macro generates an actor that owns its own `ModelContext`, making it safe to perform writes from background tasks without `@MainActor` violations. This is critical because the capture loop (Spotify polling / MusicKit observation) runs on background actors per ADR-001.

4. **Index coverage for UI queries.** `#Index` on `CaptureSession.startTime`, `CaptureSession.state`, and `CapturedTrack.playedAt` ensures that the session list, active-sessions badge, and track timeline views remain fast as history grows. Free-tier retention cleanup queries against `endTime` also benefit.

5. **Cascade delete simplifies retention.** Deleting an expired `CaptureSession` automatically removes all its `CapturedTrack` rows and associated `SyncMetadata` records via `.cascade` delete rules. No manual cleanup is needed.

## Consequences

### Positive

- **Fast append-only writes.** Track inserts during capture are single-row operations isolated to a background `ModelActor`, with no main-thread blocking.
- **Efficient retention cleanup.** A single `#Predicate` fetch + batch delete removes expired sessions and all related data in one pass, thanks to cascade rules.
- **Sync-ready from day one.** `SyncMetadata` on both sessions and tracks gives the future sync engine fine-grained dirty tracking without schema changes.
- **SwiftUI integration is trivial.** `@Query` with `#Predicate` provides reactive, filtered, sorted views with no boilerplate.
- **Collaborative sessions are structurally supported.** The `collaboratorIDs` array and per-track sync state provide the foundation for real-time multi-user sessions in Pro tier.

### Negative

- **Four model classes increase surface area.** Developers must understand the relationship graph and remember to use the `ModelActor` for writes rather than inserting directly into a view's `ModelContext`.
- **Potential N+1 on session lists.** Displaying track counts or previews in a session list requires either `relationshipKeyPathsForPrefetching` or a denormalized `trackCount` property. We choose prefetching initially and will denormalize if profiling reveals a bottleneck.
- **SyncMetadata on every track adds storage overhead.** For free-tier users who never sync, each track carries an unused `SyncMetadata` record. This is a minor cost (a few bytes per track) and avoids schema divergence between tiers.

### Risks

- **SwiftData maturity.** SwiftData is newer than Core Data and may have edge-case bugs with complex predicates or large relationship graphs. Mitigation: comprehensive unit tests using in-memory `ModelContainer`, and the `QueueifyModelContainer.create(inMemory: true)` factory makes this straightforward.
- **Migration complexity.** Adding fields to `@Model` classes in future releases requires SwiftData migration plans. Mitigation: keep models lean, use optional properties for new fields, and define `VersionedSchema` types before shipping v1.
- **Cascade delete performance.** Deleting a session with thousands of tracks could block the `ModelActor` for a non-trivial duration. Mitigation: retention cleanup runs on the background actor during app-backgrounding, never on the main thread. If needed, deletions can be batched.

## Implementation Notes

1. **ModelContainer setup.** Initialize `QueueifyModelContainer.create()` at app launch and inject it via `.modelContainer()` on the root `WindowGroup`. Pass the same container to the `CaptureModelActor` initializer.

2. **CaptureModelActor lifetime.** Create one `CaptureModelActor` per active capture session. The actor is initialized with the shared `ModelContainer` and lives for the duration of the session. It is discarded when the session completes or the user cancels.

3. **Retention cleanup scheduling.** Call `deleteExpiredSessions()` on app launch and on every `scenePhase` transition to `.background`. Gate the call on `tier == .free`.

4. **Token storage.** `ProviderAccount.accessToken` and `refreshToken` are stored in the SwiftData model for convenience during development. Before production release, migrate sensitive tokens to Keychain and store only a Keychain reference in the model. This is tracked separately and does not affect the model graph design.

5. **Sendable conformance.** All `@Model` classes are marked `final class` and declared `Sendable`. The `@Model` macro in iOS 26 supports `Sendable` conformance when all stored properties are themselves `Sendable`. Enums are `Codable & Sendable` to satisfy this requirement.

6. **Testing.** Use `QueueifyModelContainer.create(inMemory: true)` in unit tests. Inject the in-memory container into `CaptureModelActor` instances to test the full write path without touching disk.

7. **VersionedSchema.** Before the first public release, wrap all models in a `QueueifySchemaV1: VersionedSchema` and define a `SchemaMigrationPlan` so that future schema changes can be handled with lightweight or custom migrations.
