# Queueify — Product Brief

**Version:** 0.1 (Draft)
**Date:** 2026-03-31
**Status:** RFC — ready for refinement

---

## 1. Problem

Every music listener has experienced this: you're on a road trip, at a party, or deep in a flow state, and the queue of songs that played was *perfect*. But once the moment passes, that queue is gone. Streaming services let you queue music forward, but none of them let you look back and capture what just played as a reusable playlist.

Spotify's "Recently Played" view is limited, unsaveable, and loses context quickly. Apple Music doesn't surface play history at all on most surfaces. There is no cross-platform tool that solves this.

## 2. Solution

**Queueify** is a companion app (web + iOS) that listens to your active streaming session, records every track that plays, and lets you save any slice of that listening history as a playlist — back to your streaming service, shareable with friends, or exported as a list.

### Core user flow

1. Connect your Spotify (or Apple Music) account.
2. Tap "Start Capture" (or enable auto-capture).
3. Listen to music normally through your streaming app.
4. When you're ready, open Queueify and see your full session history.
5. Select tracks (or the whole session) and tap "Save as Playlist."
6. The playlist appears in your streaming app, ready to share.

## 3. Target Users

| Segment | Use case |
|---|---|
| **Road trippers** | Capture the playlist that soundtracked a 6-hour drive |
| **Party hosts / DJs** | Save the night's setlist after collaborative queuing |
| **Discovery junkies** | Record radio/autoplay sessions to revisit later |
| **Workout listeners** | Capture the mix that got them through a PR session |
| **Couples / friend groups** | Build shared playlists from joint listening sessions |

## 4. Platform & API Feasibility

### Spotify (Primary — MVP target)

Spotify's Web API provides everything we need:

| Capability | Endpoint | Notes |
|---|---|---|
| Poll current track | `GET /me/player/currently-playing` | Returns track, progress, device, timestamp |
| Recently played | `GET /me/player/recently-played` | Up to 50 tracks with `played_at` timestamps |
| Read queue | `GET /me/player/queue` | Returns upcoming queue (useful for "capture forward" feature) |
| Create playlist | `POST /me/playlists` | Writes directly to user's library (old `/users/{user_id}/playlists` removed Feb 2026) |
| Add tracks | `POST /playlists/{id}/tracks` | Up to 100 tracks per request |

**Required OAuth scopes:** `user-read-currently-playing`, `user-read-recently-played`, `user-read-playback-state`, `playlist-modify-public`, `playlist-modify-private`

**Key constraints:**
- Recently-played endpoint returns max 50 tracks — this is why we need active polling rather than relying solely on history
- Rate limits are per-app, not per-endpoint; Spotify uses a sliding window (anecdotally ~180 requests/minute per user for well-behaved apps)
- As of February 2026, Spotify requires Premium accounts for apps in "Development Mode" and limits to 5 test users; full production ("Extended Quota Mode") requires Spotify review and is only available to organizations
- Polling `currently-playing` every 5-10 seconds is the recommended capture strategy

### Apple Music (Phase 2 — iOS only)

| Capability | Method | Notes |
|---|---|---|
| Poll current track | `MusicKit ApplicationMusicPlayer.shared.queue` | Native iOS framework only |
| Recently played | `GET /v1/me/recent/played/tracks` | REST API, but limited to ~30 items, no per-play timestamps |
| Create playlist | `MusicKit` or `POST /v1/me/library/playlists` | Works on both native and REST |

**Key constraints:**
- Queue reading and reliable now-playing data require a **native iOS app** using MusicKit — the REST API alone is insufficient for real-time capture
- Apple Music on web has no equivalent to `ApplicationMusicPlayer`
- Apple's developer token (JWT) does not auto-refresh; requires server-side token management
- Practical implication: Apple Music support is iOS-native only (no web capture)

### Other Platforms (Future / Not Viable)

| Platform | Status | Reason |
|---|---|---|
| YouTube Music | No official API | Unofficial `ytmusicapi` exists but relies on cookie auth; fragile and TOS-violating |
| Amazon Music | Closed beta | API exists but is invite-only with no public timeline |
| Tidal | No key endpoints | Official API lacks currently-playing and recently-played endpoints entirely |

**Recommendation:** Launch with Spotify on web + iOS. Add Apple Music on iOS in Phase 2. Monitor YouTube Music and Tidal for API improvements.

## 5. Capture Architecture

### Strategy: Hybrid polling + history backfill

```
┌──────────────┐     poll every 5-10s      ┌──────────────────┐
│  Queueify    │ ◄─────────────────────────►│  Spotify API     │
│  Client/     │    /me/player/currently-   │                  │
│  Worker      │     playing                │                  │
└──────┬───────┘                            └──────────────────┘
       │
       │  track changed? dedupe + append
       ▼
┌──────────────┐
│  Capture     │  session_id, track_id, played_at,
│  Session DB  │  artist, album, duration, source
└──────────────┘
```

**Why polling, not webhooks?** Spotify does not offer real-time playback webhooks. Polling `currently-playing` is the standard pattern used by scrobbling apps (Last.fm, stats.fm, etc.).

**Deduplication logic:**
- Track same track ID appearing in consecutive polls (same song still playing)
- Use `progress_ms` + `timestamp` to detect replays vs. continued playback
- Backfill from `recently-played` on app open to catch anything missed while backgrounded

**Where polling runs:**
- **Web:** Client-side polling via the browser while the tab is open; optional server-side polling for "always-on" capture (premium feature)
- **iOS:** Background App Refresh + foreground polling; MusicKit observer for Apple Music

## 6. Feature Roadmap

### Phase 1 — MVP (Spotify Web)

- Spotify OAuth login
- Real-time capture via client-side polling
- Session timeline view (track list with timestamps, album art)
- "Save as Playlist" to Spotify
- Basic session management (start, stop, rename, delete)
- Responsive web app (mobile-friendly)

### Phase 2 — iOS + Polish

- Native iOS app (SwiftUI)
- Apple Music support via MusicKit
- Background capture (iOS background refresh)
- Push notifications ("Your session captured 47 tracks — save it?")
- Session sharing (link to view a session's tracklist)
- Collaborative sessions (multiple people contribute to one capture)

### Phase 3 — Social + Intelligence

- Public session feed ("What people are listening to at SXSW")
- AI-powered session naming ("Friday Night House Party")
- Session stats (genres, decades, energy arc, BPM graph)
- "Remix Session" — reorder, remove, and refine before saving
- Cross-platform playlist export (convert between Spotify/Apple Music/etc.)
- Integration with Last.fm scrobble history

## 7. Monetization

### Principles

1. **The core capture-and-save flow is always free.** Users should never pay to save a playlist from a session they just lived.
2. **Charge for power, not for access.** Premium features add convenience and depth, not gates around basic functionality.
3. **No ads. Ever.** Music listeners already tolerate enough ads on free tiers; Queueify should feel like a premium tool, not an ad platform.
4. **Transparent pricing.** No dark patterns, no surprise charges, no "contact sales."

### Free Tier

- Unlimited manual capture sessions (must have app/tab open)
- Save playlists to your streaming service
- 30-day session history
- Up to 3 active sessions at a time

### Queueify Pro — $3.99/month or $29.99/year

- **Always-on capture:** Server-side polling so you never miss a track, even when the app is closed
- **Unlimited session history:** Sessions never expire
- **Collaborative sessions:** Invite friends to a shared capture (party mode)
- **Session analytics:** Genre breakdowns, energy curves, BPM graphs, listening streaks
- **Smart playlists:** Auto-create playlists from sessions matching rules (e.g., "every Friday evening session")
- **Priority export:** Faster playlist creation, batch export, cross-platform conversion
- **Early access** to new features

### Why this pricing works

- **$3.99/month is impulse-priced** — less than a single coffee, positioned well below Spotify Premium ($11.99)
- **Annual discount (37% off)** incentivizes commitment and reduces churn
- **Server-side polling is the natural upsell** — it has real infrastructure cost and delivers real incremental value
- **No feature feels punitive to remove** — free users get the full core experience

### Revenue projections (illustrative)

| Scenario | MAU | Conversion | ARPU | MRR |
|---|---|---|---|---|
| Seed | 10,000 | 5% | $3.99 | $2,000 |
| Growth | 100,000 | 7% | $3.80 | $26,600 |
| Scale | 1,000,000 | 8% | $3.60 | $288,000 |

### Future monetization opportunities (non-scummy)

- **Queueify for DJs** — Professional tier with setlist export, BPM matching, transition notes ($9.99/month)
- **Queueify for Venues** — Dashboard for bars/venues to capture and display what's playing, share setlists with patrons, analytics on what music drives engagement ($29.99/month)
- **Affiliate revenue** — "Like this session? Here's a vinyl version" links to record stores (commission-based, clearly labeled, opt-in)
- **API access** — Let other apps build on Queueify's capture infrastructure

## 8. Technical Stack (Proposed)

| Layer | Technology | Rationale |
|---|---|---|
| **Web frontend** | Next.js (React) + Tailwind CSS | SSR for SEO, great DX, fast iteration |
| **iOS app** | SwiftUI + MusicKit | Native feel, background refresh, MusicKit integration |
| **API server** | Node.js (Fastify) or Python (FastAPI) | Lightweight, async-friendly for polling workers |
| **Database** | PostgreSQL + Redis | Postgres for sessions/users/tracks; Redis for rate-limit tracking and polling state |
| **Background jobs** | BullMQ (Node) or Celery (Python) | Server-side polling workers for Pro users |
| **Auth** | OAuth 2.0 (Spotify, Apple) + session tokens | Standard flow; no password management needed |
| **Hosting** | Vercel (web) + Railway or Fly.io (API) + Supabase (DB) | Low-ops, generous free tiers for MVP |
| **Payments** | Stripe | Industry standard, handles subscriptions, Apple Pay support |

## 9. Risks & Mitigations

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| Spotify revokes API access or changes terms | Medium | Critical | Apply for Extended Quota Mode early; maintain good standing; diversify to Apple Music |
| Spotify rate-limits aggressive polling | Medium | High | Adaptive polling interval (slow down when idle, speed up on track changes); respect `Retry-After` headers |
| Apple rejects iOS app (perceived duplication) | Low | High | Focus app review messaging on "playlist creation from history" — Apple has approved similar apps (e.g., stats.fm, SongShift) |
| Low conversion to paid | Medium | Medium | Validate willingness to pay via landing page test before building Pro features |
| Competitors copy the feature | Medium | Low | Move fast, build community, establish brand; streaming services adding this natively is the real risk — and also validation |
| Privacy concerns (tracking listening) | Low | Medium | Clear consent flows, easy data deletion, privacy-first messaging, no data sold to third parties |

## 10. Success Metrics

| Phase | Metric | Target |
|---|---|---|
| MVP launch | Sessions captured | 1,000 in first month |
| MVP launch | Playlists saved | 500 in first month |
| MVP launch | D7 retention | >30% |
| Growth | MAU | 10,000 within 6 months |
| Growth | Pro conversion | >5% of MAU |
| Growth | NPS | >50 |

## 11. Open Questions

1. **Should we support "retroactive capture"?** Spotify's recently-played gives the last 50 tracks — we could offer a "rescue" flow even if the user wasn't capturing. How prominent should this be?
2. **Collaborative queue capture:** If multiple people at a party are connected, should we merge their capture streams or keep them separate?
3. **Scrobbling integration:** Many power users already use Last.fm. Should we pull from scrobble history as an alternative data source?
4. **Offline playback:** Spotify doesn't report offline plays until the device reconnects. How do we message this gap to users?
5. **Naming:** Is "Queueify" the right name? Alternatives considered: Setlist, Requeue, Playback, Traceback, SessionSnap.

---

## Next Steps

1. Refine this brief based on team feedback
2. Write ADRs for key technical decisions:
   - ADR-001: Polling strategy and interval tuning
   - ADR-002: Client-side vs. server-side capture architecture
   - ADR-003: Data model for sessions and tracks
   - ADR-004: Authentication and multi-provider OAuth
   - ADR-005: Monetization and subscription infrastructure
3. Write PCDs (Product Concept Documents) for each phase
4. Build landing page to validate demand before writing code
5. Register for Spotify Extended Quota Mode
