# TDD-09: Guilds & Clans

> **Project:** Ultimate Game Engine — Multiplayer Game Server  
> **Technical Design:** Guilds & Clans  
> **Version:** 1.0  
> **Last Updated:** 2026-07-01  
> **Status:** Draft  
> **Priority:** Technical Architecture

---

## 1. Purpose & Scope

Define the requirements for a persistent guild/clan system enabling players to form long-lived organizations with membership management, roles, permissions, chat, and metadata. Guilds provide a social structure beyond friends and parties, supporting cooperative gameplay and community building.

---

Refer to [BRD-09](../BRD/09_guilds_clans.md) for the business requirements and [PRD-09](../PRD/09_guilds_clans.md) for the API surface.

---

## 2. Architecture & Design Flow

The guild/clan system is persistent and relies on PostgreSQL. Relationships between players and groups are stored in `group_edge` using bidirectional edge pairs (Group $\to$ User, User $\to$ Group).

### Guild Application & Acceptance Sequence
```mermaid
sequenceDiagram
    autonumber
    actor Player as Applicant (Player A)
    participant Server as Server Core
    participant DB as PostgreSQL DB
    actor Admin as Guild Admin

    Player->>Server: POST /v2/group/{id}/join
    Server->>DB: Insert invite edge User A -> Group (state=3, join_request)
    DB-->>Server: Confirmation
    Server-->>Admin: (WebSocket) Emit join_request event
    
    Admin->>Server: POST /v2/group/{id}/accept/{user_id}
    Server->>DB: Start Transaction:<br/>Update edges state=2 (member)<br/>Increment edge_count in 'groups'<br/>Verify edge_count <= max_count
    DB-->>Server: Transaction Commit
    Server-->>Player: (WebSocket) Emit 'guild_joined' notification
```

---

## 3. Database Schema & Data Models

### Raw DDL Schemas

```sql
CREATE TABLE IF NOT EXISTS groups (
    id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    creator_id       UUID REFERENCES users(id) ON DELETE SET NULL,
    name             VARCHAR(128) NOT NULL,
    description      VARCHAR(1024),
    avatar_url       VARCHAR(512),
    lang_tag         VARCHAR(18) DEFAULT 'en-US' NOT NULL,
    open             BOOLEAN DEFAULT TRUE NOT NULL,
    edge_count       INT DEFAULT 0 NOT NULL,
    max_count        INT DEFAULT 100 NOT NULL,
    metadata         JSONB DEFAULT '{}'::jsonb NOT NULL,
    state            SMALLINT DEFAULT 0 NOT NULL, -- 0=normal, 1=deleted
    create_time      TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP NOT NULL,
    update_time      TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP NOT NULL,
    disable_time     TIMESTAMPTZ
);

CREATE TABLE IF NOT EXISTS group_edge (
    source_id        UUID NOT NULL, -- user_id or group_id
    destination_id   UUID NOT NULL, -- group_id or user_id
    state            SMALLINT NOT NULL, -- 0=superadmin, 1=admin, 2=member, 3=join_request
    position         BIGINT DEFAULT (EXTRACT(EPOCH FROM NOW()) * 1000)::bigint NOT NULL,
    update_time      TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP NOT NULL,
    PRIMARY KEY (source_id, state, destination_id)
);
```

### Table Indexes

```sql
-- Enforce unique group names for active groups
CREATE UNIQUE INDEX IF NOT EXISTS idx_groups_name_active 
ON groups (name) 
WHERE state = 0;

-- Index for listing members of a group sorted by role and join date
CREATE INDEX IF NOT EXISTS idx_group_edge_membership
ON group_edge (source_id, state, position DESC);
```

---

## 4. Algorithmic Logic & Execution Flow

### Role-Based Privilege Hierarchies
Operations check user credentials before modifying edges:
- **Demote / Promote**: Only users with state `0` (superadmin) can promote/demote admins (`1`). Admins can promote members (`2`) to admins.
- **Kick**: Superadmin can kick anyone. Admins can kick members (`2`). Members cannot kick.

### SQL Transaction for Request Acceptance

```sql
BEGIN;

-- 1. Check member capacity constraints
SELECT edge_count, max_count FROM groups WHERE id = $1 FOR UPDATE;

-- 2. Update edge from Group -> User
INSERT INTO group_edge (source_id, destination_id, state, position)
VALUES ($1, $2, 2, (EXTRACT(EPOCH FROM NOW())*1000)::bigint)
ON CONFLICT (source_id, state, destination_id) DO UPDATE SET state = 2, update_time = NOW();

-- 3. Update edge from User -> Group
INSERT INTO group_edge (source_id, destination_id, state, position)
VALUES ($2, $1, 2, (EXTRACT(EPOCH FROM NOW())*1000)::bigint)
ON CONFLICT (source_id, state, destination_id) DO UPDATE SET state = 2, update_time = NOW();

-- 4. Update overall membership counter
UPDATE groups SET edge_count = edge_count + 1 WHERE id = $1;

COMMIT;
```

---

## 6. Performance & Security Considerations

### Performance
- **Edge Count Reconciliation**: Run a periodic background job (every 10 minutes) that reconciles `groups.edge_count` with the actual `COUNT(*)` of `group_edge` rows for each group. Log discrepancies and auto-correct.
- **FOR UPDATE Lock Contention**: The `SELECT ... FOR UPDATE` on the `groups` row during join acceptance can bottleneck for popular guilds. Consider using `SKIP LOCKED` or advisory locks to reduce contention.
- **Max Guild Size**: Enforce `max_count` (default 100). If a guild's `edge_count >= max_count`, reject joins atomically within the transaction.
- **Pagination**: Guild member lists and guild search results must be paginated (max 100 per page, cursor-based).
- **Latency Target**: Guild join/leave transactions p99 <50ms.

### Security
- **Role-Based Authorization**: Enforce privilege hierarchy strictly on the server side:
  - Only `state=0` (superadmin) can delete the guild, promote/demote admins, and transfer ownership.
  - Only `state≤1` (admin+) can kick members, accept join requests, and edit guild metadata.
  - `state=2` (member) can only read guild info and leave voluntarily.
- **Guild Name Sanitization**: Validate guild names: max 128 characters, strip leading/trailing whitespace, reject names containing only whitespace or control characters.
- **Metadata Size Limit**: `metadata` JSONB max **4 KB** per guild. Reject oversized payloads.
- **Abuse Prevention**:
  - Rate limit guild creation: max **3 guilds created per user per day**.
  - Rate limit join requests: max **5 join requests per user per hour** across all guilds.
- **Soft Delete Protection**: Deleting a guild sets `state=1` but retains data. Ensure deleted guilds are excluded from all search queries and edge lookups.

---

## 5. Linked Documents
- [BRD-09](../BRD/09_guilds_clans.md) (Business Requirements Document)
- [PRD-09](../PRD/09_guilds_clans.md) (Product Requirements Document)
