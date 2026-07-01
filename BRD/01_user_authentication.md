# BRD-01: User Authentication

> **Project:** Ultimate Game Engine — Multiplayer Game Server  
> **Feature:** User Authentication  
> **Version:** 1.0  
> **Last Updated:** 2026-07-01  
> **Status:** Draft  
> **Priority:** Must Have
---
## 1. Purpose

Define the requirements for a comprehensive user authentication system that supports multiple identity providers, session management, and account linking. This system serves as the foundational identity layer for all other game server features.
---
## 2. Scope

### In Scope
- Multi-provider authentication (email, device, social, platform, custom)
- User account creation and management
- Session token issuance and validation
- Refresh token mechanism
- Account linking (multiple identities per user)
- JWT-based authentication
- Rate limiting on authentication endpoints

### Out of Scope
- Authorization / role-based access control (covered by individual feature BRDs)
- Payment provider integration
- Two-factor authentication (future consideration)
---
## 3. Stakeholders

| Role | Interest |
|------|----------|
| Game Developers | Easy integration with game engines, multiple auth providers |
| Players | Seamless login, cross-platform identity |
| Server Administrators | Security, monitoring, user management |
| Security Team | Token security, identity verification, anti-abuse |
---
## 4. Functional Requirements

### FR-AUTH-001: Email/Password Authentication
- **Priority:** Must
- Users shall register with email and password.
- Passwords shall be hashed using bcrypt (or Argon2id) with a minimum work factor.
- Email validation format shall be enforced.
- Optional: Email verification flow.

### FR-AUTH-002: Device ID Authentication
- **Priority:** Must
- Users shall authenticate using a unique device identifier.
- An account shall be automatically created on first use.
- Device IDs shall be treated as opaque strings.
- Support for device migration (linking device to existing account).

### FR-AUTH-003: Social Authentication — Apple
- **Priority:** Should
- Support Apple Sign In via Apple Identity Token.
- Extract and store Apple user ID and email (if provided).
- Handle Apple's private relay email addresses.

### FR-AUTH-004: Social Authentication — Google
- **Priority:** Should
- Support Google Sign-In via OAuth 2.0 / Google ID token.
- Extract and store Google user ID, email, and display name.

### FR-AUTH-005: Social Authentication — Facebook
- **Priority:** Should
- Support Facebook Login via Facebook access token.
- Extract and store Facebook user ID, email, and display name.

### FR-AUTH-006: Platform Authentication — Steam
- **Priority:** Should
- Support Steam authentication via Steam session ticket.
- Validate ticket against Steam Web API.
- Extract and store Steam ID and persona name.

### FR-AUTH-007: Platform Authentication — Game Center
- **Priority:** Could
- Support Apple Game Center authentication.
- Validate Game Center credentials server-side.

### FR-AUTH-008: Social Authentication — Facebook Instant Games
- **Priority:** Should
- Support authentication using Facebook Instant Games signed player signatures.
- Authenticate the user and retrieve user profile info.
- Handle automatic user account creation on first sign-in.

### FR-AUTH-009: Custom Authentication
- **Priority:** Must
- Support custom authentication via a developer-defined identifier and optional secret.
- Allow integration with external identity providers not natively supported.
- Custom auth IDs shall be namespaced to prevent collision.

### FR-AUTH-010: JWT Authentication
- **Priority:** Must
- Issue JWT tokens upon successful authentication.
- Tokens shall contain: user ID, username, expiry, issued-at, and custom claims.
- Token signing shall use RS256 or HS256 (configurable).
- Token expiry shall be configurable (default: 60 minutes).

### FR-AUTH-011: Session Tokens
- **Priority:** Must
- A session token shall be issued upon successful authentication.
- Session tokens shall be required for all authenticated API calls.
- Session tokens shall have a configurable TTL.
- The server shall validate session tokens on every request.

### FR-AUTH-012: Refresh Tokens
- **Priority:** Must
- A refresh token shall be issued alongside the session token.
- Refresh tokens shall have a longer TTL than session tokens (configurable, default: 7 days).
- The refresh endpoint shall issue a new session token and optionally rotate the refresh token.
- Revoked refresh tokens shall be rejected.

### FR-AUTH-013: Account Linking
- **Priority:** Must
- Users shall be able to link multiple identity providers to a single account.
- Linking shall require an active session.
- Unlinking an identity provider shall be supported (minimum one identity must remain).
- Conflict resolution: if a provider identity is already linked to another account, return an error.

### FR-AUTH-014: User Accounts
- **Priority:** Must
- Each user shall have a unique user ID (UUID v4).
- User profile fields: `id`, `username`, `display_name`, `avatar_url`, `lang`, `location`, `timezone`, `metadata`, `create_time`, `update_time`.
- Username shall be unique and optional (auto-generated if not provided).
- Display name shall not need to be unique.
- User metadata shall be a JSON object (max 16 KB).

### FR-AUTH-015: Session Revocation
- **Priority:** Should
- Support explicit session logout / token revocation.
- Support server-initiated session invalidation (e.g., ban).
- Optional: revoke all sessions for a user.

### FR-AUTH-016: Rate Limiting
- **Priority:** Must
- Authentication endpoints shall enforce rate limiting.
- Configurable limits per IP and per user.
- Failed authentication attempts shall be tracked.
- Account lockout after N failed attempts (configurable).
---
## 5. Non-Functional Requirements

| Requirement | Specification |
|-------------|---------------|
| **Performance** | Authentication requests shall complete within 200ms (p99) |
| **Scalability** | Support 10,000+ concurrent authentication requests |
| **Security** | Passwords hashed with bcrypt/Argon2id; tokens signed with RS256/HS256 |
| **Availability** | 99.9% uptime for authentication endpoints |
| **Compliance** | GDPR-compliant data handling for user PII |
| **Logging** | All authentication events shall be logged (success, failure, linking) |
| **Monitoring** | Metrics for auth success/failure rates, latency, and token usage |
---
## 6. Dependencies

| Dependency | Purpose |
|------------|---------|
| PostgreSQL | User data storage |
| JWT Library | Token generation and validation |
| bcrypt/Argon2id | Password hashing |
| OAuth 2.0 | Social provider integration |
| Steam Web API | Steam ticket validation |
| Apple Auth API | Apple Sign In validation |
---
## 7. Acceptance Criteria

- [ ] Users can register and authenticate via email/password
- [ ] Users can authenticate via device ID with auto-account creation
- [ ] Users can authenticate via at least 3 social/platform providers
- [ ] Custom authentication works with arbitrary identifiers
- [ ] JWT session tokens are issued with configurable expiry
- [ ] Refresh tokens can be used to obtain new session tokens
- [ ] Multiple identities can be linked to a single account
- [ ] Identities can be unlinked (minimum one must remain)
- [ ] Rate limiting prevents brute-force attacks
- [ ] All auth events are logged
- [ ] p99 latency ≤ 200ms for authentication requests
---
## 8. Risks & Mitigations

| Risk | Impact | Likelihood | Mitigation |
|------|--------|------------|------------|
| Social provider API changes | Auth broken for affected provider | Medium | Abstract provider layer; version SDKs |
| Token theft | Account compromise | Medium | Short-lived tokens; refresh rotation; HTTPS only |
| Password database breach | User credentials exposed | Low | bcrypt/Argon2id hashing; per-user salts |
| Rate limiting bypass (distributed) | Brute-force attacks | Medium | IP + user combined limits; CAPTCHA fallback |
| Device ID spoofing | Account takeover | Medium | Treat device auth as low-trust; encourage linking |
---
## 9. Future Considerations

- Two-factor authentication (TOTP, SMS)
- Passkey / WebAuthn support
- OAuth 2.0 / OpenID Connect server (act as identity provider)
- Account deletion workflow (GDPR right to erasure)
- Account merge (combine two accounts)
- IP-based geolocation for security alerts
---
## Version History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2026-07-01 | Engineering | Initial BRD |

---

## Linked Documents
- [PRD-01](../PRD/01_user_authentication.md) (API Surface & Interface Specification)
- [TDD-01](../TDD/01_user_authentication.md) (Database Schema & Technical Implementation Design)
