# 22. Console Admin

## 1. Overview

The Console Admin module provides a RESTful API and backend logic to support the Developer Dashboard. It allows authenticated team members to monitor the server, manage game entities, and provide customer support securely.

## 2. Features and APIs

### 2.1 Authentication & Sessions
- **POST `/console/authenticate`**: Authenticate via username/email and password. Returns a secure, HTTP-only session cookie or an admin JWT.
- **POST `/console/logout`**: Invalidate the current console session.

### 2.2 Account Management (Admin Users)
- **GET `/console/users`**: List all console users (Admins only).
- **POST `/console/users`**: Create a new console user.
- **DELETE `/console/users/{id}`**: Delete a console user.

### 2.3 Player Data Inspection
- **GET `/console/api/users`**: Search game players by ID, username, or email.
- **GET `/console/api/users/{id}`**: Fetch detailed player profile, including wallet, metadata, and connected devices.
- **POST `/console/api/users/{id}/ban`**: Ban a player (disables login).
- **POST `/console/api/users/{id}/tombstone`**: Soft-delete a player (GDPR compliance).
- **GET `/console/api/users/{id}/notes`**: Retrieve admin notes left on a player's profile.
- **POST `/console/api/users/{id}/notes`**: Add an administrative note to a player.

### 2.4 Economy & Wallets
- **GET `/console/api/users/{id}/wallet/ledger`**: View a player's transaction history.
- **POST `/console/api/users/{id}/wallet`**: Grant or deduct virtual currency from a player's wallet.

### 2.5 Realtime Server Inspection
- **GET `/console/api/matches`**: List active authoritative matches across the cluster.
- **GET `/console/api/status`**: Fetch server node health metrics, memory usage, and connected client counts.

### 2.6 Audit Logging
- **GET `/console/api/audit`**: View the immutable audit log of administrative actions, filtered by admin user or target resource.

## 3. Security Considerations
- **Network Shielding:** The Console API must run on a separate port (`7351`) and should be restricted to internal networks or VPNs.
- **Fine-Grained Access Control:** Permissions are evaluated using a JSONB configuration (`acl`) specifying allowed resource actions (e.g. `{"admin": true}` or `{"write_players": true}`). Simple integer roles are not used.
- **Multi-Factor Authentication (MFA):** Supports TOTP-based MFA. If enabled, users must supply a valid authentication code.
- **Immutable Audit Logging:** All mutating endpoints (`POST`, `PUT`, `DELETE`) executed by a console operator must asynchronously append records to `console_audit_log`. These records must be immutable.
