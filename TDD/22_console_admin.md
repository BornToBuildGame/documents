# 22. Console Admin

## 1. Architecture

The Console Admin system runs as an integrated service within the Ultimate Game Engine nodes but exposes its REST API via a dedicated HTTP multiplexer, ensuring it can be restricted to internal networks or VPNs. 

Admin authentication utilizes secure, signed JWTs distinct from player JWTs. Session tokens are embedded in `HttpOnly` cookies to prevent XSS attacks.

## 2. Database Schema Integration

The Console Admin leverages the following tables defined in `database-design.md`:

- **`console_user`**: Stores admin credentials, utilizing Bcrypt for password hashing.
- **`console_audit_log`**: An append-only table recording every administrative action (who, what, when, and the payload).
- **`console_acl_template`**: Defines the JSON-based permission templates mapped to roles (Admin, Developer, Maintainer, ReadOnly).
- **`setting`**: Key-Value store for global server configurations dynamically modifiable by admins.
- **`users_notes`**: Customer support notes tied to player UUIDs.

## 3. Role-Based Access Control (RBAC)

Each endpoint is protected by middleware that evaluates the authenticated user's role against the `console_acl_template`.

- **Role 1 (Admin):** Full system access.
- **Role 2 (Developer):** Can view all data and modify game state (wallets, leaderboards) but cannot manage `console_users`.
- **Role 3 (Maintainer):** Can view server metrics and restart services; cannot view PII (Personally Identifiable Information).
- **Role 4 (ReadOnly):** Can view player profiles and metrics; cannot mutate any state.

## 4. Auditing Middleware

All administrative endpoints (except pure `GET` requests) are wrapped in an Audit Middleware. 
Upon successful execution of a request, the middleware asynchronously writes an entry to `console_audit_log` containing the target resource ID and the request payload.

## 5. Implementation Details

- **Language/Framework:** Go `net/http` with strict routing.
- **Authentication:** JWTs signed with a console-specific secret key (distinct from the player session key).
- **Search Capabilities:** The player search endpoint utilizes ILIKE indexing on `users.username` and `users.email` with a hard limit of 50 results to prevent slow queries on large datasets.
