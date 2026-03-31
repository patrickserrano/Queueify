# ADR-007: Subscription Infrastructure — RevenueCat + StoreKit 2

**Status:** Accepted
**Date:** 2026-03-31
**Deciders:** Queueify Engineering

## Context

Queueify offers two tiers: a free tier and a Pro tier ($3.99/month or $29.99/year). The free tier provides unlimited manual capture, 30-day session history, and up to 3 active sessions. Pro unlocks always-on server-side polling, unlimited session history, collaborative sessions, session analytics, smart playlists, priority export, and cross-platform conversion.

We need infrastructure to:

1. **Sell subscriptions** through the App Store on iOS.
2. **Gate Pro features** on-device so the UI immediately reflects the user's entitlement status.
3. **Verify subscription status server-side** so the API server can authorize server-side polling, collaborative session creation, and other Pro-only backend operations.
4. **Support future platforms.** An Android app is planned, and a web dashboard is under consideration. The subscription system must eventually work across all three surfaces without requiring users to re-purchase.

The app targets iOS 26+ with Swift 6 strict concurrency. All concurrency uses `async/await`, `AsyncSequence`, and `@Observable`. Combine is prohibited.

Apple takes a 30% commission on App Store transactions (15% under the Small Business Program for developers earning under $1M/year). This commission structure affects pricing and margin calculations for the Pro tier.

## Decision

We will use **RevenueCat SDK + StoreKit 2** as the subscription infrastructure on iOS. RevenueCat serves as the server-side source of truth for entitlement status and provides cross-platform subscription management. On-device, we additionally listen to StoreKit 2's `Transaction.updates` async sequence for real-time status changes. Stripe integration is deferred until the Android/web launch.

## Options Considered

### Option A: Raw StoreKit 2 only

Use Apple's StoreKit 2 framework directly for all subscription management. Handle receipt validation, entitlement tracking, subscription lifecycle (renewals, grace periods, billing retry, refunds), and server-side verification ourselves using App Store Server API and App Store Server Notifications V2.

**Pros:**
- No third-party dependency or additional SDK cost.
- Full control over every aspect of the subscription lifecycle.
- StoreKit 2's modern Swift API (`Product`, `Transaction`, `Transaction.updates`) is well-designed and async/await-native.

**Cons:**
- Significant engineering effort to implement server-side receipt validation, subscription status polling, and webhook handling for App Store Server Notifications V2.
- No built-in cross-platform support. When Android launches, we would need to build an entirely separate Google Play Billing integration and a custom server to unify entitlements across platforms.
- No built-in analytics for subscription metrics (MRR, churn, trial conversion, cohort analysis).
- Must manually handle edge cases: grace periods, billing retry states, family sharing entitlements, offer code redemption, subscription upgrades/downgrades/crossgrades.
- Ongoing maintenance burden as Apple evolves the App Store Server API.

### Option B: RevenueCat + StoreKit 2

Use RevenueCat's SDK on the client, which wraps StoreKit 2 under the hood. RevenueCat's backend handles receipt validation, entitlement management, cross-platform identity, and webhook delivery. On-device, we also observe StoreKit 2's `Transaction.updates` for instant status changes (RevenueCat's SDK does this internally, but we maintain our own listener for redundancy and immediate UI updates).

**Pros:**
- Dramatically less code to write and maintain. RevenueCat handles receipt validation, subscription lifecycle, grace periods, billing retry, and refund detection.
- Cross-platform support out of the box. When Android launches, we add the RevenueCat Android SDK and entitlements unify automatically under the same RevenueCat project. Same for Stripe on web.
- Server-side webhooks. RevenueCat sends webhook events (INITIAL_PURCHASE, RENEWAL, CANCELLATION, EXPIRATION, BILLING_ISSUE, etc.) to our API server, enabling server-side entitlement verification without polling the App Store.
- Built-in subscription analytics: MRR, churn rate, trial conversion, LTV, cohort analysis — available from day one without building a data pipeline.
- Pricing experiments and paywall A/B testing via RevenueCat's remote configuration (Offerings/Paywalls).
- Free for apps under $2.5K MTR; 1% of tracked revenue after that.

**Cons:**
- Third-party dependency. If RevenueCat has an outage, subscription checks degrade (mitigated by local caching and StoreKit 2 fallback).
- Additional SDK in the binary (~2 MB).
- 1% revenue share above the $2.5K MTR threshold (on top of Apple's 15-30%).

### Option C: Stripe only (web-based billing)

Bypass the App Store entirely by directing users to a web-based checkout powered by Stripe. The app would check subscription status via our own API server.

**Pros:**
- No Apple commission (Stripe charges ~2.9% + $0.30 per transaction vs. Apple's 15-30%).
- Full control over the billing experience.
- Works on all platforms from day one.

**Cons:**
- Apple's App Store guidelines require in-app purchase for digital goods and services consumed within the app. Directing users to an external payment flow for unlocking in-app features violates the guidelines and risks app rejection.
- While recent regulatory changes (EU Digital Markets Act, US court rulings) have created narrow exceptions for linking out, compliance is uncertain and region-dependent. Building on this as the primary iOS billing path is premature.
- Stripe has no native iOS SDK for subscription management — we would need to build the entire client-side purchase flow, entitlement checking, and webhook handling ourselves.
- Worse user experience: users expect to pay with their Apple ID and have subscriptions managed in Settings.

## Rationale

**Option B (RevenueCat + StoreKit 2) is the clear choice** for the following reasons:

1. **Cross-platform is the deciding factor.** Queueify's roadmap includes an Android app. Without RevenueCat (or a similar abstraction), we would need to build and maintain two completely separate billing integrations (StoreKit 2 + Google Play Billing) and a custom server to unify user entitlements. RevenueCat solves this with a single SDK per platform and a unified backend.

2. **Server-side webhooks are essential.** Queueify Pro includes always-on server-side polling — the server must know whether a user has an active Pro subscription to run their polling jobs. RevenueCat's webhooks deliver subscription lifecycle events directly to our API server, avoiding the need to build and maintain our own App Store Server Notifications V2 integration.

3. **Reduced engineering surface.** Subscription billing has a long tail of edge cases (grace periods, billing retry, ask-to-buy, family sharing, offer codes, refunds, subscription transfers). RevenueCat handles all of these. Building this from scratch with raw StoreKit 2 would consume weeks of engineering time with high ongoing maintenance cost.

4. **Analytics from day one.** Understanding MRR, churn, and trial conversion is critical for a subscription business. RevenueCat provides these metrics without building a data pipeline.

5. **Stripe is not viable as a primary iOS billing mechanism** due to App Store guidelines. It remains the right choice for a future web dashboard, and RevenueCat integrates with Stripe for exactly this scenario.

## Consequences

### Positive

- **Fast time to market.** Subscription infrastructure can be implemented in days rather than weeks.
- **Cross-platform ready.** Adding Android subscriptions later requires only adding the RevenueCat Android SDK and configuring Google Play credentials — no server-side changes.
- **Reliable server-side verification.** RevenueCat webhooks ensure the API server always has up-to-date subscription status for server-side polling authorization.
- **Resilient on-device checking.** RevenueCat caches entitlement status locally. Combined with our own StoreKit 2 `Transaction.updates` listener, the app can gate Pro features even when offline.
- **Subscription analytics available immediately** without custom instrumentation.

### Negative

- **Third-party dependency.** A RevenueCat outage could delay subscription status updates (mitigated by local caching and StoreKit 2 fallback).
- **Revenue share.** RevenueCat charges 1% of tracked revenue above $2.5K MTR. At scale, this adds up (e.g., $100K ARR = ~$1K/year to RevenueCat, on top of Apple's commission).
- **SDK size.** RevenueCat adds ~2 MB to the app binary.
- **Abstraction leakage.** If we need StoreKit 2 behavior that RevenueCat doesn't expose, we may need to work around the SDK or file a feature request.

### Risks

- **RevenueCat pricing changes.** RevenueCat could raise its revenue share or change its free tier threshold. Mitigated: the `SubscriptionManager` abstraction means we could swap to raw StoreKit 2 on iOS without changing the rest of the app.
- **Webhook delivery delays.** RevenueCat webhook delivery is near-real-time but not instant. If the server needs to authorize a Pro action and the webhook hasn't arrived yet, we fall back to a synchronous RevenueCat REST API check.
- **App Store commission changes.** Apple's 15% Small Business Program rate is not guaranteed long-term. Pricing may need adjustment if the commission structure changes.

## Implementation Notes

### SubscriptionManager (@Observable)

The `SubscriptionManager` is the single source of truth for subscription status on-device. It conforms to `@Observable` for SwiftUI integration and wraps RevenueCat's `Purchases` SDK. It also maintains a direct StoreKit 2 `Transaction.updates` listener for redundancy.

```swift
import RevenueCat
import StoreKit
import Observation

enum SubscriptionTier: Sendable {
    case free
    case pro
}

@Observable
@MainActor
final class SubscriptionManager {
    private(set) var tier: SubscriptionTier = .free
    private(set) var expirationDate: Date?
    private(set) var isInGracePeriod: Bool = false
    private(set) var isInBillingRetry: Bool = false

    /// Whether the user has an active Pro entitlement (including grace period).
    var isPro: Bool {
        tier == .pro
    }

    private var transactionListenerTask: Task<Void, Never>?

    init() {
        // Configure RevenueCat on app launch
        Purchases.logLevel = .warn
        Purchases.configure(
            with: .init(withAPIKey: Configuration.revenueCatAPIKey)
                .with(appUserID: nil) // Anonymous until login
        )

        // Start listening for StoreKit 2 transaction updates
        transactionListenerTask = Task { [weak self] in
            for await result in Transaction.updates {
                guard let self else { break }
                if case .verified(let transaction) = result {
                    await transaction.finish()
                    await self.refreshEntitlements()
                }
            }
        }

        // Initial entitlement check
        Task {
            await refreshEntitlements()
        }
    }

    deinit {
        transactionListenerTask?.cancel()
    }

    /// Fetch the latest entitlement status from RevenueCat.
    func refreshEntitlements() async {
        do {
            let customerInfo = try await Purchases.shared.customerInfo()
            updateFromCustomerInfo(customerInfo)
        } catch {
            // On failure, retain the last known state (cached locally by RevenueCat)
        }
    }

    /// Purchase a package (monthly or annual).
    func purchase(_ package: Package) async throws {
        let result = try await Purchases.shared.purchase(package: package)

        if !result.userCancelled {
            updateFromCustomerInfo(result.customerInfo)
        }
    }

    /// Restore purchases (e.g., after reinstall or device transfer).
    func restorePurchases() async throws {
        let customerInfo = try await Purchases.shared.restorePurchases()
        updateFromCustomerInfo(customerInfo)
    }

    /// Identify the user after authentication (links anonymous to known user ID).
    func identify(userID: String) async throws {
        let (customerInfo, _) = try await Purchases.shared.logIn(userID)
        updateFromCustomerInfo(customerInfo)
    }

    /// Fetch available offerings (for displaying the paywall).
    func fetchOfferings() async throws -> Offerings {
        try await Purchases.shared.offerings()
    }

    // MARK: - Private

    private func updateFromCustomerInfo(_ info: CustomerInfo) {
        let proEntitlement = info.entitlements["pro"]

        if let pro = proEntitlement, pro.isActive {
            tier = .pro
            expirationDate = pro.expirationDate
            // RevenueCat tracks grace period and billing retry states
            // via the entitlement's periodType and ownership info.
        } else {
            tier = .free
            expirationDate = nil
        }
    }
}
```

### Gating Pro Features in SwiftUI

Views gate Pro features by reading from the `SubscriptionManager` in the environment:

```swift
struct SessionListView: View {
    @Environment(SubscriptionManager.self) private var subscriptionManager

    var body: some View {
        List {
            // All users see their sessions
            ForEach(sessions) { session in
                SessionRow(session: session)
            }

            // Pro-only: session analytics
            if subscriptionManager.isPro {
                Section("Analytics") {
                    SessionAnalyticsView()
                }
            }
        }
        .toolbar {
            // Pro-only: collaborative sessions
            if subscriptionManager.isPro {
                Button("Invite Collaborator") {
                    showCollaboratorSheet = true
                }
            }
        }
    }
}

// Enforce free-tier session limit
struct NewSessionButton: View {
    @Environment(SubscriptionManager.self) private var subscriptionManager
    let activeSessionCount: Int

    private let freeSessionLimit = 3

    var body: some View {
        Button("New Session") {
            startNewSession()
        }
        .disabled(
            !subscriptionManager.isPro && activeSessionCount >= freeSessionLimit
        )
    }
}
```

### RevenueCat Webhook to API Server

RevenueCat sends webhook events to our API server whenever a subscription lifecycle event occurs. The server uses this to maintain its own `user_subscriptions` table for authorizing server-side operations (e.g., always-on polling).

**Webhook endpoint:** `POST /api/webhooks/revenuecat`

```
RevenueCat Event Types We Handle:
- INITIAL_PURCHASE       -> Create/activate subscription record
- RENEWAL                -> Extend subscription expiration
- CANCELLATION           -> Mark subscription as canceled (still active until period end)
- EXPIRATION             -> Deactivate subscription, stop server-side polling
- BILLING_ISSUE_DETECTED -> Flag user, send retention notification
- SUBSCRIBER_ALIAS       -> Link anonymous user to authenticated user
- PRODUCT_CHANGE         -> Handle upgrade/downgrade between monthly and annual
```

The server validates the webhook by checking the `Authorization` header against a shared secret configured in RevenueCat's webhook settings. On each event, the server updates the user's subscription record and starts/stops server-side polling jobs accordingly.

For cases where the webhook has not yet arrived (e.g., user just purchased and immediately requests a Pro feature via the API), the server falls back to a synchronous check against RevenueCat's REST API:

```
GET https://api.revenuecat.com/v1/subscribers/{app_user_id}
Authorization: Bearer {REVENUECAT_API_KEY}
```

### Server-Side Entitlement Check Flow

```
Client requests Pro feature (e.g., start server-side polling)
  |
  v
API server checks local `user_subscriptions` table
  |
  +-- Record exists and is active? -> Authorize the request
  |
  +-- No record or expired? -> Synchronous check against RevenueCat REST API
        |
        +-- Active entitlement? -> Update local record, authorize the request
        |
        +-- No active entitlement? -> Return 403 Forbidden
```

### Pricing Strategy

| Plan | Price | Effective Monthly (User) | Apple Commission (15% SBP) | RevenueCat (1%) | Net Revenue/Month |
|---|---|---|---|---|---|
| Monthly | $3.99/mo | $3.99 | $0.60 | $0.04 | $3.35 |
| Annual | $29.99/yr | $2.50 | $0.37 | $0.02 | $2.11 |

The annual plan represents a 37% discount vs. monthly, incentivizing longer commitment and reducing churn. The annual plan has a lower net revenue per month but significantly higher LTV due to reduced churn.

**Introductory offers:**
- 7-day free trial on both monthly and annual plans (configured in App Store Connect and mapped to RevenueCat Offerings).
- Trial eligibility is managed by StoreKit 2 / App Store — each Apple ID gets one trial per subscription group.

**Family Sharing:**
- Enabled for Pro subscriptions via App Store Connect's Family Sharing setting.
- RevenueCat tracks family-shared entitlements and reports them distinctly in analytics (family members do not count as separate paying subscribers).
- The server-side webhook includes family sharing metadata, so the API server can authorize family members for server-side polling.

### Apple Small Business Program

Queueify will enroll in Apple's Small Business Program, which reduces the App Store commission from 30% to 15% for developers earning less than $1M in annual proceeds. At Queueify's early stage, this is a given. The pricing table above assumes the 15% rate.

If Queueify's annual proceeds exceed $1M, the commission reverts to 30% for the following calendar year, and pricing should be reevaluated. RevenueCat's analytics dashboard provides real-time proceeds tracking to monitor this threshold.

### RevenueCat Configuration

| RevenueCat Concept | Queueify Mapping |
|---|---|
| **App** | Queueify iOS (initially); Queueify Android added later |
| **Entitlement** | `pro` — single entitlement for all Pro features |
| **Product** | `queueify_pro_monthly` ($3.99), `queueify_pro_annual` ($29.99) |
| **Offering** | `default` — contains both monthly and annual packages |
| **App User ID** | Anonymous until login, then identified with Queueify user ID via `Purchases.shared.logIn()` |

### Stripe (Deferred)

Stripe integration is deferred until the Android app or web dashboard launches. When that time comes:

- RevenueCat natively integrates with Stripe as a billing provider. Stripe purchases will flow into the same RevenueCat entitlement system, so a user who subscribes on web via Stripe and later installs the iOS app will see their Pro entitlement on both platforms.
- The same webhook endpoint and server-side entitlement table will work for Stripe-originated subscriptions — RevenueCat normalizes events across all billing providers.
- Stripe's lower transaction fees (~2.9% + $0.30) make web-based subscriptions more profitable per subscriber, but the additional engineering and support cost should be weighed against the actual demand for a web billing surface.

### Migration Path (If RevenueCat Is Removed)

The `SubscriptionManager` abstraction isolates the rest of the app from RevenueCat. If we ever need to remove RevenueCat (cost, outage concerns, or capability gaps), the migration path is:

1. Replace `SubscriptionManager` internals with direct StoreKit 2 `Product.purchase()` and `Transaction.currentEntitlements` calls.
2. Replace the RevenueCat webhook with App Store Server Notifications V2.
3. Replace the RevenueCat REST API fallback with App Store Server API calls.
4. The rest of the app (views, server-side entitlement table, feature gating) remains unchanged.

This migration is estimated at 1-2 weeks of engineering effort, primarily in server-side receipt validation logic.
