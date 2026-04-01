# ADR-004: Authentication — Spotify OAuth 2.0 PKCE + MusicKit Authorization

**Status:** Accepted
**Date:** 2026-03-31
**Deciders:** Queueify Engineering

## Context

Queueify captures music listening sessions from Spotify and Apple Music and saves them as playlists. Users can connect one or both services — there are no app-specific passwords. All authentication is delegated to provider OAuth or on-device authorization flows.

The two providers have fundamentally different auth models:

- **Spotify** uses OAuth 2.0 Authorization Code with PKCE. Access tokens expire after 1 hour and must be refreshed using a refresh token. Required scopes are `user-read-currently-playing`, `user-read-recently-played`, `user-read-playback-state`, `playlist-modify-public`, and `playlist-modify-private`. As of February 2026, Spotify's Dev Mode requires a Premium account and limits the app to 5 test users until a quota extension is approved.
- **Apple Music** uses MusicKit's on-device authorization. A developer token (JWT) is signed server-side and bundled or fetched by the app. A Music User Token is obtained on-device via `MusicAuthorization.request()`. There is no refresh flow — when the token expires, the app re-authorizes.

For Pro-tier users, Queueify performs server-side polling of the Spotify API on the user's behalf, which requires the server to hold a valid Spotify access token (and refresh token) for each connected user.

The codebase enforces modern Swift conventions: iOS 26, Swift 6+, structured concurrency with async/await throughout. Combine, completion handlers, singletons, and direct GCD usage are all prohibited.

## Decision

Implement a unified `AuthManager` (an `@Observable @MainActor` class injected via the SwiftUI environment) that owns two independent auth-state machines — one for Spotify OAuth 2.0 PKCE and one for MusicKit. Tokens are stored in the iOS Keychain. Spotify tokens are synced to the server for Pro server-side polling. All async work uses Swift structured concurrency.

## Options Considered

### Option A: Unified AuthManager with per-provider state machines (Chosen)

A single `@Observable @MainActor` `AuthManager` holds a `SpotifyAuthState` and a `MusicKitAuthState`, each modeled as an enum (`.disconnected`, `.authorized(...)`, `.expired`, `.failed(Error)`). The manager exposes async methods for login, logout, and token refresh. Tokens live in the Keychain. The manager is injected via `.environment()` in the SwiftUI app root.

### Option B: Separate manager classes per provider

A `SpotifyAuthManager` and `MusicKitAuthManager` each independently manage their provider. This increases surface area, complicates coordinated UI state (e.g., an "accounts" screen showing both connection statuses), and doubles the injection points.

### Option C: Server-mediated OAuth for Spotify (no PKCE, use server-side code exchange)

The server holds the client secret and exchanges the authorization code on behalf of the client. This avoids PKCE on the device but introduces a server dependency for all users (not just Pro) and requires the client secret to live on the server, increasing the attack surface.

## Rationale

- **Option A** keeps auth state co-located and easy to observe from SwiftUI views. A single environment object means any view can read both connection states without additional plumbing.
- PKCE on-device (Option A) is the recommended flow for native apps per RFC 7636 and Spotify's own documentation. It avoids exposing a client secret entirely — the app never possesses one.
- MusicKit authorization is inherently on-device; wrapping it in the same manager is lightweight and consistent.
- Option B was rejected because the two providers' states are frequently shown together (e.g., settings screen, onboarding) and splitting them adds unnecessary coordination logic.
- Option C was rejected because it forces a server round-trip for free-tier users who have no need for server-side token storage, and it requires safeguarding a client secret on the server.

## Consequences

### Positive

- **Single source of truth** — all auth state is observable from one `@Observable` object in the SwiftUI environment.
- **No client secret** — PKCE eliminates the need for a Spotify client secret on-device or on-server.
- **Keychain security** — tokens are stored in the iOS Keychain with `kSecAttrAccessibleAfterFirstUnlock`, using iOS Data Protection via the keybag (available after first device unlock after boot). Stronger Secure Enclave–backed policies (e.g., biometry/user presence) can be added with appropriate Keychain access control flags if needed.
- **Pro server-side polling** — Spotify tokens are synced to the backend only for Pro users, keeping free-tier data entirely on-device.
- **Testable** — `AuthManager` can be initialized with a `TokenStore` protocol and an `AuthSessionProvider` protocol, allowing full unit testing with mocked Keychain and mocked web auth sessions.

### Negative

- **Spotify Dev Mode limitation** — during development, only 5 Premium test users can authenticate until a quota extension is granted. This constrains beta testing.
- **Token refresh complexity** — the app must handle token refresh transparently before every Spotify API call, including edge cases like concurrent refresh attempts and refresh token rotation.
- **MusicKit re-authorization** — since MusicKit has no refresh flow, the user may see a system prompt to re-authorize periodically, which is outside the app's control.

### Risks

- **Refresh token revocation** — if a user revokes access via Spotify's account settings, the refresh token becomes invalid. The app must detect `401` responses, clear stored tokens, transition to `.disconnected`, and prompt the user to reconnect.
- **Keychain migration** — if the Keychain item attributes change across app versions, a migration path is needed to avoid locking users out.
- **Server token leakage** — Spotify tokens stored server-side for Pro polling are high-value targets. They must be encrypted at rest and scoped to the minimum required permissions.

## Implementation Notes

### Spotify OAuth 2.0 PKCE Flow

1. Generate a cryptographically random `code_verifier` (43-128 characters, unreserved URI characters).
2. Derive `code_challenge` = Base64URL(SHA256(`code_verifier`)).
3. Open `ASWebAuthenticationSession` pointing to `https://accounts.spotify.com/authorize` with query parameters: `client_id`, `response_type=code`, `redirect_uri`, `scope`, `code_challenge_method=S256`, `code_challenge`.
4. On callback, extract the `code` from the redirect URI.
5. Exchange the `code` + `code_verifier` for an access token and refresh token via `POST https://accounts.spotify.com/api/token`.
6. Store both tokens in the Keychain. Update `SpotifyAuthState` to `.authorized`.
7. For Pro users, send the tokens to the Queueify backend over TLS for server-side polling.

### Token Refresh

Before any Spotify API call, check whether the access token is expired (track the `expires_in` timestamp). If expired, use the refresh token to obtain a new access token. If the refresh fails with an invalid-grant error, transition to `.disconnected` and surface a reconnect prompt.

Handle concurrent refresh: use a Swift `actor` or a continuation-based lock so that multiple concurrent API calls awaiting a refresh only trigger one network request, and all callers receive the new token.

### MusicKit Authorization Flow

1. Call `await MusicAuthorization.request()`.
2. On `.authorized`, the system provides a Music User Token automatically to MusicKit API calls.
3. Store the authorization status in `MusicKitAuthState`. No tokens are manually managed — MusicKit handles this internally.
4. If authorization status changes to `.denied` or `.restricted`, transition to `.disconnected`.

### Auth State Model

```swift
enum SpotifyAuthState: Sendable {
    case disconnected
    case authorized(accessToken: String, expiresAt: Date)
    case expired          // access token expired, refresh token still available
    case failed(AuthError)
}

enum MusicKitAuthState: Sendable {
    case disconnected
    case authorized
    case denied
    case failed(AuthError)
}
```

### AuthManager Pseudocode

```swift
import AuthenticationServices
import MusicKit

protocol TokenStore: Sendable {
    func save(key: String, data: Data) async throws
    func load(key: String) async throws -> Data?
    func delete(key: String) async throws
}

protocol AuthSessionProvider: Sendable {
    func authenticate(url: URL, callbackScheme: String) async throws -> URL
}

@Observable
@MainActor
final class AuthManager {
    private(set) var spotifyState: SpotifyAuthState = .disconnected
    private(set) var musicKitState: MusicKitAuthState = .disconnected

    private let tokenStore: TokenStore
    private let sessionProvider: AuthSessionProvider
    private let spotifyClientID: String
    private let spotifyRedirectURI: String

    var isSpotifyConnected: Bool {
        if case .authorized = spotifyState { return true }
        return false
    }

    var isMusicKitConnected: Bool {
        if case .authorized = musicKitState { return true }
        return false
    }

    init(
        tokenStore: TokenStore,
        sessionProvider: AuthSessionProvider,
        spotifyClientID: String,
        spotifyRedirectURI: String
    ) {
        self.tokenStore = tokenStore
        self.sessionProvider = sessionProvider
        self.spotifyClientID = spotifyClientID
        self.spotifyRedirectURI = spotifyRedirectURI
    }

    // MARK: - Spotify

    func connectSpotify() async throws {
        let verifier = PKCEUtil.generateCodeVerifier()
        let challenge = PKCEUtil.generateCodeChallenge(from: verifier)

        let authURL = SpotifyEndpoints.authorizeURL(
            clientID: spotifyClientID,
            redirectURI: spotifyRedirectURI,
            challenge: challenge,
            scopes: [
                "user-read-currently-playing",
                "user-read-recently-played",
                "user-read-playback-state",
                "playlist-modify-public",
                "playlist-modify-private"
            ]
        )

        let callbackURL = try await sessionProvider.authenticate(
            url: authURL,
            callbackScheme: "queueify"
        )

        guard let code = URLComponents(url: callbackURL, resolvingAgainstBaseURL: false)?
            .queryItems?.first(where: { $0.name == "code" })?.value else {
            throw AuthError.missingAuthorizationCode
        }

        let tokenResponse = try await SpotifyAPI.exchangeCode(
            code: code,
            verifier: verifier,
            clientID: spotifyClientID,
            redirectURI: spotifyRedirectURI
        )

        try await tokenStore.save(
            key: "spotify_access_token",
            data: Data(tokenResponse.accessToken.utf8)
        )
        try await tokenStore.save(
            key: "spotify_refresh_token",
            data: Data(tokenResponse.refreshToken.utf8)
        )

        let expiresAt = Date.now.addingTimeInterval(
            TimeInterval(tokenResponse.expiresIn)
        )
        spotifyState = .authorized(
            accessToken: tokenResponse.accessToken,
            expiresAt: expiresAt
        )
    }

    func refreshSpotifyTokenIfNeeded() async throws -> String {
        guard case .authorized(let token, let expiresAt) = spotifyState else {
            if case .expired = spotifyState {
                return try await performSpotifyTokenRefresh()
            }
            throw AuthError.notConnected
        }

        if expiresAt < Date.now {
            spotifyState = .expired
            return try await performSpotifyTokenRefresh()
        }

        return token
    }

    private func performSpotifyTokenRefresh() async throws -> String {
        guard let refreshData = try await tokenStore.load(key: "spotify_refresh_token"),
              let refreshToken = String(data: refreshData, encoding: .utf8) else {
            spotifyState = .disconnected
            throw AuthError.missingRefreshToken
        }

        do {
            let tokenResponse = try await SpotifyAPI.refreshToken(
                refreshToken: refreshToken,
                clientID: spotifyClientID
            )

            try await tokenStore.save(
                key: "spotify_access_token",
                data: Data(tokenResponse.accessToken.utf8)
            )
            if let newRefresh = tokenResponse.refreshToken {
                try await tokenStore.save(
                    key: "spotify_refresh_token",
                    data: Data(newRefresh.utf8)
                )
            }

            let expiresAt = Date.now.addingTimeInterval(
                TimeInterval(tokenResponse.expiresIn)
            )
            spotifyState = .authorized(
                accessToken: tokenResponse.accessToken,
                expiresAt: expiresAt
            )
            return tokenResponse.accessToken
        } catch AuthError.invalidGrant {
            spotifyState = .disconnected
            try await tokenStore.delete(key: "spotify_access_token")
            try await tokenStore.delete(key: "spotify_refresh_token")
            throw AuthError.tokenRevoked
        }
    }

    func disconnectSpotify() async throws {
        try await tokenStore.delete(key: "spotify_access_token")
        try await tokenStore.delete(key: "spotify_refresh_token")
        spotifyState = .disconnected
    }

    // MARK: - MusicKit

    func connectMusicKit() async {
        let status = await MusicAuthorization.request()
        switch status {
        case .authorized:
            musicKitState = .authorized
        case .denied:
            musicKitState = .denied
        case .restricted:
            musicKitState = .failed(.restricted)
        default:
            musicKitState = .disconnected
        }
    }

    func disconnectMusicKit() {
        musicKitState = .disconnected
    }

    // MARK: - Restore Session

    func restoreSession() async {
        // Spotify
        if let tokenData = try? await tokenStore.load(key: "spotify_access_token"),
           let token = String(data: tokenData, encoding: .utf8) {
            // Attempt a lightweight API call to verify the token
            spotifyState = .expired  // assume expired, refresh will correct
            _ = try? await refreshSpotifyTokenIfNeeded()
        }

        // MusicKit
        let musicStatus = MusicAuthorization.currentStatus
        if musicStatus == .authorized {
            musicKitState = .authorized
        }
    }
}
```

### Token Sync to Server (Pro Users)

After a successful Spotify token exchange or refresh, Pro users sync the encrypted tokens to the Queueify backend:

```swift
func syncTokensToServer() async throws {
    guard case .authorized(let accessToken, _) = spotifyState else { return }
    guard let refreshData = try await tokenStore.load(key: "spotify_refresh_token"),
          let refreshToken = String(data: refreshData, encoding: .utf8) else { return }

    try await QueueifyAPI.syncSpotifyTokens(
        accessToken: accessToken,
        refreshToken: refreshToken
    )
}
```

The server stores these tokens encrypted at rest and uses them to poll Spotify on the user's behalf. When the server refreshes a token, it notifies the client via push notification so the local Keychain stays in sync.

### Keychain Storage

Use a `KeychainTokenStore` conforming to `TokenStore`. Store tokens with:
- `kSecAttrAccessibleAfterFirstUnlock` — tokens are available after the first device unlock, enabling background refresh.
- `kSecAttrService` set to the app's bundle identifier to namespace entries.
- `kSecAttrSynchronizable` set to `false` — tokens must not sync to iCloud Keychain.

### Security Considerations

- **No client secret on-device.** PKCE eliminates the need entirely for public clients.
- **TLS for all network calls.** App Transport Security enforces this by default on iOS.
- **Keychain, not UserDefaults.** Tokens are never stored in UserDefaults, files, or Core Data.
- **Short-lived access tokens.** Spotify access tokens expire in 1 hour, limiting the window if one is compromised.
- **Refresh token rotation.** If Spotify returns a new refresh token during a refresh, the old one is replaced immediately.
- **Server-side encryption.** Tokens stored on the backend for Pro polling are encrypted at rest using AES-256-GCM.
- **Minimal scopes.** Only the scopes required for reading playback state and writing playlists are requested.
- **PKCE verifier entropy.** The code verifier is generated using `SecRandomCopyBytes` with at least 32 bytes of entropy, Base64URL-encoded.
