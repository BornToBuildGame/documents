# BRD-21: In-App Purchase (IAP) Validation

> **Project:** Ultimate Game Engine — Multiplayer Game Server  
> **Feature:** In-App Purchase Validation  
> **Version:** 1.0  
> **Last Updated:** 2026-07-01  
> **Status:** Draft  
> **Priority:** Must Have  
> **Reference:** [Nakama IAP Validation](https://heroiclabs.com/docs/nakama/concepts/iap-validation/)
---
## 1. Purpose

Define the requirements for a built-in server-side in-app purchase (IAP) validation system. This system validates purchase receipts from Apple App Store, Google Play, Huawei AppGallery, and Facebook Instant Games, preventing fraud through replay attack detection and receipt verification. This is a **built-in feature** of the game server, not a custom implementation.
---
## 2. Scope

### In Scope
- Server-side receipt validation for Apple App Store
- Server-side receipt validation for Google Play
- Server-side receipt validation for Huawei AppGallery
- Server-side receipt validation for Facebook Instant Games
- Single product purchase validation
- Subscription purchase validation and lifecycle tracking
- Replay attack prevention (`seenBefore` tracking)
- Real-time provider notifications (refunds, subscription state changes)
- Purchase and subscription history per user
- Unity IAP receipt routing
- Notification hooks for purchase/subscription state changes

### Out of Scope
- Payment processing (handled by platform providers)
- Virtual currency / wallet management (see BRD-13: Economy System)
- Pricing configuration
- Store listing management
---
## 3. Stakeholders

| Role | Interest |
|------|----------|
| Game Developers | Secure purchase validation, easy integration |
| Players | Fair purchasing, subscription management |
| Monetization Team | Revenue integrity, fraud prevention |
| Finance/Legal | Audit trail, refund tracking, compliance |
---
## 4. Functional Requirements

### FR-IAP-001: Apple App Store Validation
- **Priority:** Must
- Validate Apple purchase receipts against Apple's verification servers.
- Support single product purchases and subscriptions.
- Support iOS 7+ receipt format.
- Support Apple StoreKit 2 (JWS representation).
- Handle Apple real-time server notifications for:
  - Refunds
  - Subscription renewals
  - Subscription cancellations
  - Subscription expirations
- Endpoint: `POST /v2/iap/purchase/apple`
- Subscription validation: `POST /v2/iap/subscription/apple`

### FR-IAP-002: Google Play Validation
- **Priority:** Must
- Validate Google Play purchase tokens against Google Play Developer API.
- Support single product purchases and subscriptions.
- Requires Google Cloud Service Account with Play Console permissions.
- Handle Google Real-time Developer Notifications for:
  - Refunds
  - Subscription renewals
  - Subscription cancellations
  - Subscription expirations
- Endpoint: `POST /v2/iap/purchase/google`
- Subscription validation: `POST /v2/iap/subscription/google`

### FR-IAP-003: Huawei AppGallery Validation
- **Priority:** Should
- Validate Huawei purchases against Huawei IAP validation service.
- Verify purchase data signature locally before contacting Huawei's servers.
- Reject purchases with invalid signatures before remote validation.
- Requires both purchase data and signature from Huawei IAP SDK.
- Endpoint: `POST /v2/iap/purchase/huawei`

### FR-IAP-004: Facebook Instant Games Validation
- **Priority:** Could
- Validate Facebook Instant Games purchases via signed request.
- Verify signed request with Facebook's servers.
- Endpoint: `POST /v2/iap/purchase/facebook_instant`

### FR-IAP-005: Replay Attack Prevention
- **Priority:** Must
- Every validated purchase shall be stored in the database.
- On validation, check if the receipt has been validated before.
- Return a `seen_before` boolean flag:
  - `false`: first-time validation — safe to grant rewards
  - `true`: previously validated — potential replay attack
- Purchases are attached to the user ID from the session used during validation.
- If an account is deleted, purchases are associated with a system user to prevent reuse.

### FR-IAP-006: Purchase History
- **Priority:** Must
- Store all validated purchases per user.
- Purchase record fields:
  - `user_id`: the user who validated the purchase
  - `product_id`: SKU / product identifier
  - `transaction_id`: unique transaction ID from the provider
  - `store`: platform (Apple, Google, Huawei, Facebook)
  - `purchase_time`: when the purchase was made
  - `seen_before`: whether this was a repeat validation
  - `provider_response`: raw response from the provider
  - `environment`: production or sandbox
  - `refund_time`: set if the purchase was refunded
  - `create_time`: when the record was created
- List purchases by user with pagination.

### FR-IAP-007: Subscription Management
- **Priority:** Must
- Support subscription validation with additional lifecycle tracking.
- Subscription record fields (in addition to purchase fields):
  - `original_transaction_id`: the original subscription transaction
  - `expiry_time`: when the subscription expires
  - `active`: whether the subscription is currently active
  - `provider_notification`: raw provider notification data
- Get subscription by product ID.
- List all subscriptions for a user.

### FR-IAP-008: Provider Notifications
- **Priority:** Must
- Receive and process real-time server-to-server notifications from providers.
- Normalize notifications into 5 categories:

| Type | Description | Applies To |
|------|-------------|------------|
| `SUBSCRIBED` | Initial subscription or resubscription | Subscriptions |
| `RENEWED` | Subscription auto-renewed | Subscriptions |
| `EXPIRED` | Subscription expired, will not renew | Subscriptions |
| `CANCELLED` | Subscription cancelled by user | Subscriptions |
| `REFUNDED` | Purchase or subscription refunded | Both |

- Automatically update purchase/subscription state in database.
- Notifications only process receipts previously validated through the server.

### FR-IAP-009: Notification Hooks
- **Priority:** Should
- Server runtime hooks for purchase/subscription events:
  - `on_purchase_notification`: fires after purchase state change (refund)
  - `on_subscription_notification`: fires after subscription state change
- Hooks receive:
  - Updated purchase/subscription record
  - Notification type
  - Provider payload (raw notification data)
- Use cases:
  - Revoke player access on refund
  - Send re-engagement messages on cancellation
  - Update analytics
  - Trigger in-game notifications

### FR-IAP-010: Unity IAP Integration
- **Priority:** Should
- Support Unity IAP receipt format.
- Route Unity IAP receipts to the appropriate platform validation based on store type:
  - `GooglePlay` → Google validation
  - `AppleAppStore` → Apple validation
- Extract `Payload` from Unity IAP receipt for validation.
- Support Unity IAP 5.x with Apple StoreKit 2 (`jwsRepresentation`).
---
## 5. Non-Functional Requirements

| Requirement | Specification |
|-------------|---------------|
| **Latency** | Validation ≤ 2000ms (includes provider round-trip) |
| **Security** | All validation server-side; no client-side trust |
| **Durability** | All purchases persisted to PostgreSQL |
| **Fraud Prevention** | Replay attack detection on every validation |
| **Availability** | 99.9% uptime for validation endpoints |
| **Auditability** | Complete purchase history with provider responses |
---
## 6. Dependencies

| Dependency | Purpose |
|------------|---------|
| BRD-01: User Authentication | User identity for purchase ownership |
| BRD-13: Economy System | Granting rewards after validation |
| BRD-11: Notifications | Notifying users of subscription changes |
| BRD-18: Server Runtime & Hooks | Notification hooks |
| BRD-14: RPC & Custom APIs | Custom purchase logic |
| PostgreSQL | Purchase and subscription persistence |
| Apple App Store API | Apple receipt verification |
| Google Play Developer API | Google receipt verification |
| Huawei IAP Service | Huawei purchase verification |
---
## 7. Acceptance Criteria

- [ ] Apple purchases are validated server-side and stored
- [ ] Google purchases are validated server-side and stored
- [ ] Huawei purchases are validated with signature check and stored
- [ ] `seen_before` flag correctly identifies replay attempts
- [ ] Purchase history is queryable per user with pagination
- [ ] Subscription lifecycle is tracked (active, expired, cancelled, refunded)
- [ ] Real-time notifications from Apple and Google update purchase state
- [ ] Notification hooks fire on purchase/subscription state changes
- [ ] Deleted accounts' purchases are reassigned to system user
- [ ] Unity IAP receipts are routed to correct provider validation
---
## 8. Risks & Mitigations

| Risk | Impact | Likelihood | Mitigation |
|------|--------|------------|------------|
| Provider API changes | Broken validation | Medium | Abstract provider layer; monitor changelogs |
| Network latency to provider | Slow validation | Medium | Timeouts; retry logic; async validation option |
| Receipt fraud (jailbroken devices) | Revenue loss | Medium | Server-side only validation; seenBefore tracking |
| Notification delivery failures | Stale subscription state | Low | Periodic subscription refresh; reconciliation |
| Multi-platform receipt format differences | Implementation complexity | Medium | Normalized response format; per-provider adapters |
---
## 9. Configuration

```yaml
iap:
  apple:
    shared_password: ""            # Apple shared secret
    notifications_endpoint_id: ""  # Apple notification endpoint
  google:
    service_account: ""            # Path to service account JSON
    package_name: ""               # Google Play package name
    notifications_endpoint_id: ""  # Google RTDN endpoint
  huawei:
    public_key: ""                 # Huawei public key for signature verification
    client_id: ""                  # Huawei OAuth client ID
    client_secret: ""              # Huawei OAuth client secret
  facebook_instant:
    app_secret: ""                 # Facebook app secret
```
---
## Version History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2026-07-01 | Engineering | Initial BRD |

---

## Linked Documents
- [PRD-21](../PRD/21_iap_validation.md) (API Surface & Interface Specification)
- [TDD-21](../TDD/21_iap_validation.md) (Database Schema & Technical Implementation Design)
