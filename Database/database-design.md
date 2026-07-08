# Database Design

> **Project:** Ultimate Game Engine — Multiplayer Game Server  
> **Last Updated:** 2026-07-08  
> **Status:** Draft

Entity relationships live in [`ERD.md`](./ERD.md).

---

## Enum Types

### ticket_status

Source: [TDD-02](../TDD/02_multiplayer_matchmaking.md)

```
queued
matched
cancelled
expired
```

Used by: `matchmaking_ticket.status`

### match_status

Source: [TDD-02](../TDD/02_multiplayer_matchmaking.md)

```
pending
active
completed
cancelled
```

Used by: `matchmaker_match.status`

---

## Table Definitions

### Identity & Access

#### users

Source: [TDD-01](../TDD/01_user_authentication.md)

```
id              UUID            PRIMARY KEY DEFAULT gen_random_uuid()
username        VARCHAR(64)     UNIQUE NOT NULL
display_name    VARCHAR(128)
avatar_url      VARCHAR(512)
lang_tag        VARCHAR(18)     DEFAULT 'en-US'
location        VARCHAR(64)
timezone        VARCHAR(64)
metadata        JSONB           DEFAULT '{}'::jsonb NOT NULL
wallet          JSONB           DEFAULT '{}'::jsonb NOT NULL
email           VARCHAR(255)
password        BYTEA                                               -- Bcrypt hash
apple_id        VARCHAR(128)
google_id       VARCHAR(128)
facebook_id     VARCHAR(128)
steam_id        VARCHAR(128)
gamecenter_id   VARCHAR(128)
custom_id       VARCHAR(128)
create_time     TIMESTAMPTZ     DEFAULT CURRENT_TIMESTAMP NOT NULL
update_time     TIMESTAMPTZ     DEFAULT CURRENT_TIMESTAMP NOT NULL
verify_time     TIMESTAMPTZ
disable_time    TIMESTAMPTZ
```

Notes:
- `wallet` stores virtual currency balances as a JSONB map (e.g., `{"coins": 500, "gems": 10}`). See [TDD-13](../TDD/13_economy_system.md) for economy system details.
- Social provider ID columns (`apple_id`, `google_id`, etc.) support account linking across multiple identity providers.

#### user_sessions

Source: [TDD-01](../TDD/01_user_authentication.md)

```
token           VARCHAR(256)    PRIMARY KEY
user_id         UUID            NOT NULL REFERENCES users(id) ON DELETE CASCADE
refresh_token   VARCHAR(256)    NOT NULL
created_at      TIMESTAMPTZ     DEFAULT CURRENT_TIMESTAMP NOT NULL
expires_at      TIMESTAMPTZ     NOT NULL
revoked         BOOLEAN         DEFAULT FALSE NOT NULL
```

---

### Multiplayer & Matchmaking

#### matchmaking_ticket

Source: [TDD-02](../TDD/02_multiplayer_matchmaking.md)

```
ticket_id           UUID                PRIMARY KEY DEFAULT gen_random_uuid()
user_id             UUID                NOT NULL REFERENCES users(id) ON DELETE CASCADE
queue_name          VARCHAR(64)         NOT NULL
skill_rating        DOUBLE PRECISION    DEFAULT 1000.0 NOT NULL
region              VARCHAR(32)         NOT NULL
properties          JSONB               DEFAULT '{}'::jsonb NOT NULL
party_members       UUID[]              DEFAULT '{}'::uuid[] NOT NULL
min_count           INT                 NOT NULL
max_count           INT                 NOT NULL
count_multiple      INT
reverse_precision   BOOLEAN             DEFAULT FALSE NOT NULL
status              ticket_status       DEFAULT 'queued' NOT NULL
created_at          TIMESTAMPTZ         DEFAULT CURRENT_TIMESTAMP NOT NULL
expires_at          TIMESTAMPTZ         NOT NULL
matched_at          TIMESTAMPTZ
```

Notes:
- `party_members` stores an array of user UUIDs for party-based matchmaking.
- `reverse_precision` enables bidirectional compatibility checks during matching.

#### matchmaker_match

Source: [TDD-02](../TDD/02_multiplayer_matchmaking.md)

```
match_id        UUID            PRIMARY KEY DEFAULT gen_random_uuid()
queue_name      VARCHAR(64)     NOT NULL
players         JSONB           NOT NULL                            -- Array of objects: [{"user_id": "...", "username": "..."}]
match_token     VARCHAR(256)    NOT NULL
properties      JSONB           DEFAULT '{}'::jsonb NOT NULL
region          VARCHAR(32)     NOT NULL
created_at      TIMESTAMPTZ     DEFAULT CURRENT_TIMESTAMP NOT NULL
status          match_status    DEFAULT 'pending' NOT NULL
```

#### match_metadata

Source: [TDD-03](../TDD/03_realtime_multiplayer.md)

```
match_id        UUID            PRIMARY KEY
label           VARCHAR(512)    DEFAULT '{}'::varchar NOT NULL
player_count    INT             DEFAULT 0 NOT NULL
max_size        INT             DEFAULT 16 NOT NULL
authoritative   BOOLEAN         DEFAULT FALSE NOT NULL
node            VARCHAR(64)     NOT NULL
created_at      TIMESTAMPTZ     DEFAULT CURRENT_TIMESTAMP NOT NULL
ended_at        TIMESTAMPTZ
```

Notes:
- Active matches are primarily held in-memory. This table serves as a persistent registry for match listing and queries.
- `node` identifies the server node hosting the match.

---

### Competitive

#### leaderboard

Source: [TDD-05](../TDD/05_leaderboards.md)

```
id              VARCHAR(128)    PRIMARY KEY
sort_order      INT             DEFAULT 1 NOT NULL                  -- 0=ascending, 1=descending
operator        INT             DEFAULT 0 NOT NULL                  -- 0=best, 1=set, 2=increment
reset_schedule  VARCHAR(64)                                         -- Cron expression
metadata        JSONB           DEFAULT '{}'::jsonb NOT NULL
authoritative   BOOLEAN         DEFAULT FALSE NOT NULL
category        INT             DEFAULT 0 NOT NULL
create_time     TIMESTAMPTZ     DEFAULT CURRENT_TIMESTAMP NOT NULL
```

#### leaderboard_record

Source: [TDD-05](../TDD/05_leaderboards.md)

```
leaderboard_id  VARCHAR(128)    NOT NULL REFERENCES leaderboard(id) ON DELETE CASCADE
owner_id        UUID            NOT NULL REFERENCES users(id) ON DELETE CASCADE
username        VARCHAR(64)     NOT NULL
score           BIGINT          NOT NULL
subscore        BIGINT          DEFAULT 0 NOT NULL
num_score       INT             DEFAULT 1 NOT NULL                  -- Number of submissions
metadata        JSONB           DEFAULT '{}'::jsonb NOT NULL
create_time     TIMESTAMPTZ     DEFAULT CURRENT_TIMESTAMP NOT NULL
update_time     TIMESTAMPTZ     DEFAULT CURRENT_TIMESTAMP NOT NULL
expiry_time     TIMESTAMPTZ                                         -- For reset intervals
PRIMARY KEY (leaderboard_id, owner_id)
```

Notes:
- Also used by the Tournament system (TDD-06). Tournament scores are scoped via `expiry_time`.
- Performance: For leaderboards >100K records, around-player queries using `DENSE_RANK()` degrade. Consider Redis Sorted Sets or split range queries.

#### tournament

Source: [TDD-06](../TDD/06_tournaments.md)

```
id              VARCHAR(128)    PRIMARY KEY
title           VARCHAR(256)    NOT NULL
description     TEXT
category        INT             DEFAULT 0 NOT NULL
sort_order      INT             DEFAULT 1 NOT NULL                  -- 0=ascending, 1=descending
operator        INT             DEFAULT 0 NOT NULL                  -- 0=best, 1=set, 2=increment
duration        INT             NOT NULL                            -- Occurrence duration in seconds
reset_schedule  VARCHAR(64)                                         -- Cron expression
start_time      TIMESTAMPTZ     DEFAULT CURRENT_TIMESTAMP NOT NULL
end_time        TIMESTAMPTZ                                         -- Total tournament end
max_size        INT             DEFAULT 0 NOT NULL                  -- Max players per sub-leaderboard (0=infinite)
max_num_score   INT             DEFAULT 0 NOT NULL                  -- Max score submissions per player
metadata        JSONB           DEFAULT '{}'::jsonb NOT NULL
authoritative   BOOLEAN         DEFAULT FALSE NOT NULL
start_active    TIMESTAMPTZ                                         -- Start of current active occurrence
create_time     TIMESTAMPTZ     DEFAULT CURRENT_TIMESTAMP NOT NULL
```

Notes:
- Tournament scoring is backed by `leaderboard_record` (see [TDD-05](../TDD/05_leaderboards.md)).
- `duration` and `reset_schedule` define repeating occurrence windows.

---

### Social

#### user_edge

Source: [TDD-07](../TDD/07_friends_system.md)

```
source_id       UUID            NOT NULL REFERENCES users(id) ON DELETE CASCADE
destination_id  UUID            NOT NULL REFERENCES users(id) ON DELETE CASCADE
state           SMALLINT        NOT NULL                            -- 0=friend, 1=invite_sent, 2=invite_received, 3=blocked
position        BIGINT          DEFAULT (EXTRACT(EPOCH FROM NOW()) * 1000)::bigint NOT NULL
update_time     TIMESTAMPTZ     DEFAULT CURRENT_TIMESTAMP NOT NULL
PRIMARY KEY (source_id, state, destination_id)
```

Notes:
- Bidirectional directed adjacency list. Each friend relationship consists of two rows (A→B and B→A).
- `position` is an epoch-millisecond value used for sorting.

#### groups

Source: [TDD-09](../TDD/09_guilds_clans.md), [TDD-16](../TDD/16_groups.md)

> This table is shared between Guilds/Clans (TDD-09) and general-purpose Groups (TDD-16).
> The `category` column differentiates them (e.g., 0=guild, 2=club).

```
id              UUID            PRIMARY KEY DEFAULT gen_random_uuid()
creator_id      UUID            REFERENCES users(id) ON DELETE SET NULL
name            VARCHAR(128)    NOT NULL
description     VARCHAR(1024)
avatar_url      VARCHAR(512)
lang_tag        VARCHAR(18)     DEFAULT 'en-US' NOT NULL
open            BOOLEAN         DEFAULT TRUE NOT NULL
edge_count      INT             DEFAULT 0 NOT NULL
max_count       INT             DEFAULT 100 NOT NULL
metadata        JSONB           DEFAULT '{}'::jsonb NOT NULL
state           SMALLINT        DEFAULT 0 NOT NULL                  -- 0=normal, 1=deleted
create_time     TIMESTAMPTZ     DEFAULT CURRENT_TIMESTAMP NOT NULL
update_time     TIMESTAMPTZ     DEFAULT CURRENT_TIMESTAMP NOT NULL
disable_time    TIMESTAMPTZ
```

Notes:
- `edge_count` tracks the current number of members. Reconciled periodically by a background job.
- `state=1` represents a soft-deleted guild/group; data is retained but excluded from search and edge lookups.

#### group_edge

Source: [TDD-09](../TDD/09_guilds_clans.md), [TDD-16](../TDD/16_groups.md)

```
source_id       UUID            NOT NULL                            -- user_id or group_id
destination_id  UUID            NOT NULL                            -- group_id or user_id
state           SMALLINT        NOT NULL                            -- 0=superadmin, 1=admin, 2=member, 3=join_request
position        BIGINT          DEFAULT (EXTRACT(EPOCH FROM NOW()) * 1000)::bigint NOT NULL
update_time     TIMESTAMPTZ     DEFAULT CURRENT_TIMESTAMP NOT NULL
PRIMARY KEY (source_id, state, destination_id)
```

Notes:
- Bidirectional edges: Group→User and User→Group pairs.
- Role hierarchy: superadmin (0) > admin (1) > member (2) > join_request (3).
- Polymorphic edges: `source_id` and `destination_id` lack FK constraints. Deleting users/groups requires application-level cleanup or database triggers to prevent orphaned rows.

---

### Communication

#### channel

Source: [TDD-10](../TDD/10_chat_system.md)

```
id              UUID            PRIMARY KEY DEFAULT gen_random_uuid()
type            SMALLINT        NOT NULL                            -- 1=DM, 2=group, 3=guild, 4=party, 5=match, 6=custom
label           VARCHAR(256)                                        -- Friendly name / identifier
group_id        UUID                                                -- References groups(id) if type=3
user_id_one     UUID            REFERENCES users(id) ON DELETE SET NULL  -- if type=1
user_id_two     UUID            REFERENCES users(id) ON DELETE SET NULL  -- if type=1
metadata        JSONB           DEFAULT '{}'::jsonb NOT NULL
create_time     TIMESTAMPTZ     DEFAULT CURRENT_TIMESTAMP NOT NULL
update_time     TIMESTAMPTZ     DEFAULT CURRENT_TIMESTAMP NOT NULL
CONSTRAINT chk_channel_type CHECK (type BETWEEN 1 AND 6)
```

Notes:
- `user_id_one` and `user_id_two` are populated only for DM channels (type=1).
- `group_id` links to the `groups` table for guild/group channels (type=3).

#### message

Source: [TDD-10](../TDD/10_chat_system.md)

```
id              UUID            PRIMARY KEY DEFAULT gen_random_uuid()
channel_id      UUID            NOT NULL REFERENCES channel(id) ON DELETE CASCADE
sender_id       UUID            NOT NULL REFERENCES users(id) ON DELETE CASCADE
username        VARCHAR(64)     NOT NULL
content         JSONB           NOT NULL
code            INT             DEFAULT 0 NOT NULL                  -- Developer-defined message code
create_time     TIMESTAMPTZ     DEFAULT CURRENT_TIMESTAMP NOT NULL
update_time     TIMESTAMPTZ     DEFAULT CURRENT_TIMESTAMP NOT NULL
deleted         BOOLEAN         DEFAULT FALSE NOT NULL
```

Notes:
- Soft-delete only (`deleted = TRUE`). Hard deletion requires admin API access.
- 90-day retention policy; a background daemon purges older messages.

#### notification

Source: [TDD-11](../TDD/11_notifications.md)

```
id              UUID            PRIMARY KEY DEFAULT gen_random_uuid()
user_id         UUID            NOT NULL REFERENCES users(id) ON DELETE CASCADE
subject         VARCHAR(256)    NOT NULL
content         JSONB           DEFAULT '{}'::jsonb NOT NULL
code            INT             DEFAULT 0 NOT NULL
sender_id       UUID            REFERENCES users(id) ON DELETE SET NULL  -- NULL for system notifications
persistent      BOOLEAN         DEFAULT TRUE NOT NULL
create_time     TIMESTAMPTZ     DEFAULT CURRENT_TIMESTAMP NOT NULL
delete_time     TIMESTAMPTZ                                         -- Used for TTL expiry
```

Notes:
- `delete_time` enables TTL-based expiry via a background daemon (runs hourly).
- Max 1,000 persistent notifications per user; oldest are pruned when cap is exceeded.

---

### Data & Storage

#### storage

Source: [TDD-12](../TDD/12_storage_engine.md)

```
collection          VARCHAR(128)    NOT NULL
key                 VARCHAR(128)    NOT NULL
user_id             UUID            NOT NULL                        -- DEFAULT '00000000-0000-0000-0000-000000000000' for global
value               JSONB           DEFAULT '{}'::jsonb NOT NULL
version             VARCHAR(64)     NOT NULL                        -- MD5 hash string
permission_read     SMALLINT        DEFAULT 1 NOT NULL              -- 0=no read, 1=owner only, 2=public
permission_write    SMALLINT        DEFAULT 1 NOT NULL              -- 0=no write (server-only), 1=owner only
create_time         TIMESTAMPTZ     DEFAULT CURRENT_TIMESTAMP NOT NULL
update_time         TIMESTAMPTZ     DEFAULT CURRENT_TIMESTAMP NOT NULL
PRIMARY KEY (collection, key, user_id)
```

Notes:
- NoSQL-style document storage backed by PostgreSQL JSONB.
- Optimistic concurrency control via `version` (MD5 hash of the value).
- `user_id = '00000000-0000-0000-0000-000000000000'` denotes global (server-owned) objects.

#### match_state

Source: [TDD-15](../TDD/15_match_state_persistence.md)

```
match_id        UUID            NOT NULL
snapshot_index  INT             NOT NULL
state           BYTEA           NOT NULL                            -- Compressed binary state payload (zlib)
tick            BIGINT          DEFAULT 0 NOT NULL
players         UUID[]          NOT NULL
metadata        JSONB           DEFAULT '{}'::jsonb NOT NULL
create_time     TIMESTAMPTZ     DEFAULT CURRENT_TIMESTAMP NOT NULL
PRIMARY KEY (match_id, snapshot_index)
```

Notes:
- Temporary checkpoint storage for authoritative match state.
- Cleaned up atomically on match termination (rows deleted after history is written).
- Max compressed state size: 512 KB.

#### match_history

Source: [TDD-15](../TDD/15_match_state_persistence.md)

```
match_id        UUID            PRIMARY KEY
match_type      VARCHAR(64)     NOT NULL
players         JSONB           NOT NULL                            -- Array of player objects
winner_id       UUID
duration        INT             NOT NULL                            -- Duration in seconds
start_time      TIMESTAMPTZ     NOT NULL
end_time        TIMESTAMPTZ     NOT NULL
metadata        JSONB           DEFAULT '{}'::jsonb NOT NULL
result          JSONB           DEFAULT '{}'::jsonb NOT NULL
create_time     TIMESTAMPTZ     DEFAULT CURRENT_TIMESTAMP NOT NULL
```

Notes:
- 365-day retention policy; older records archived to cold storage.

#### player_match

Source: [TDD-15](../TDD/15_match_state_persistence.md)

```
user_id         UUID            NOT NULL REFERENCES users(id) ON DELETE CASCADE
match_id        UUID            NOT NULL REFERENCES match_history(match_id) ON DELETE CASCADE
stats           JSONB           DEFAULT '{}'::jsonb NOT NULL
outcome         VARCHAR(16)     NOT NULL                            -- win, loss, draw
score           BIGINT          DEFAULT 0 NOT NULL
PRIMARY KEY (user_id, match_id)
```

---

### Economy

#### wallet_ledger

Source: [TDD-13](../TDD/13_economy_system.md)

```
id              UUID            PRIMARY KEY DEFAULT gen_random_uuid()
user_id         UUID            NOT NULL REFERENCES users(id) ON DELETE CASCADE
changeset       JSONB           NOT NULL                            -- e.g., {"coins": 100} or {"gems": -10}
metadata        JSONB           DEFAULT '{}'::jsonb NOT NULL        -- e.g., {"reason": "quest_reward"}
create_time     TIMESTAMPTZ     DEFAULT CURRENT_TIMESTAMP NOT NULL
```

Notes:
- Append-only audit trail for all wallet mutations.
- 180-day retention policy; older records archived to cold storage.
- `metadata` must include at minimum: `source`, `reference_id`, and `timestamp`.

---

### Monetization

#### purchase

Source: [TDD-21](../TDD/21_iap_validation.md)

```
user_id         UUID            NOT NULL REFERENCES users(id) ON DELETE CASCADE
product_id      VARCHAR(256)    NOT NULL
transaction_id  VARCHAR(256)    PRIMARY KEY
store           SMALLINT        NOT NULL                            -- 0=Apple, 1=Google, 2=Huawei, 3=Facebook
raw_response    JSONB           DEFAULT '{}'::jsonb NOT NULL
purchase_time   TIMESTAMPTZ     NOT NULL
refund_time     TIMESTAMPTZ
environment     SMALLINT        DEFAULT 0 NOT NULL                  -- 0=unknown, 1=sandbox, 2=production
seen_before     BOOLEAN         DEFAULT FALSE NOT NULL
create_time     TIMESTAMPTZ     DEFAULT CURRENT_TIMESTAMP NOT NULL
update_time     TIMESTAMPTZ     DEFAULT CURRENT_TIMESTAMP NOT NULL
```

Notes:
- `seen_before` flag prevents replay attacks and duplicate reward grants.
- `refund_time` is set when a store refund notification is received, triggering a wallet clawback.

#### subscription

Source: [TDD-21](../TDD/21_iap_validation.md)

```
user_id             UUID            NOT NULL REFERENCES users(id) ON DELETE CASCADE
product_id          VARCHAR(256)    NOT NULL
original_txn_id     VARCHAR(256)    PRIMARY KEY
store               SMALLINT        NOT NULL                        -- 0=Apple, 1=Google, 2=Huawei, 3=Facebook
raw_response        JSONB           DEFAULT '{}'::jsonb NOT NULL
purchase_time       TIMESTAMPTZ     NOT NULL
expiry_time         TIMESTAMPTZ     NOT NULL
refund_time         TIMESTAMPTZ
environment         SMALLINT        DEFAULT 0 NOT NULL              -- 0=unknown, 1=sandbox, 2=production
active              BOOLEAN         DEFAULT TRUE NOT NULL
create_time         TIMESTAMPTZ     DEFAULT CURRENT_TIMESTAMP NOT NULL
update_time         TIMESTAMPTZ     DEFAULT CURRENT_TIMESTAMP NOT NULL
```

---

### Infrastructure

#### schema_version

Source: [TDD-20](../TDD/20_database_infrastructure.md)

```
version         VARCHAR(64)     PRIMARY KEY
migration_time  TIMESTAMPTZ     DEFAULT CURRENT_TIMESTAMP NOT NULL
```

Notes:
- Tracks database migration versions. Checked at server boot to determine if migration scripts need to run.

---

## Index Strategy

### Identity & Access

#### IDX_users_email

```
UNIQUE INDEX ON users(email) WHERE email IS NOT NULL
```

Purpose: Social/provider lookup by email. Partial unique index allows multiple rows where email is NULL.

#### IDX_users_apple_id

```
UNIQUE INDEX ON users(apple_id) WHERE apple_id IS NOT NULL
```

Purpose: Apple identity provider lookup.

#### IDX_users_google_id

```
UNIQUE INDEX ON users(google_id) WHERE google_id IS NOT NULL
```

Purpose: Google identity provider lookup.

#### IDX_users_facebook_id

```
UNIQUE INDEX ON users(facebook_id) WHERE facebook_id IS NOT NULL
```

Purpose: Facebook identity provider lookup.

#### IDX_users_steam_id

```
UNIQUE INDEX ON users(steam_id) WHERE steam_id IS NOT NULL
```

Purpose: Steam identity provider lookup.

#### IDX_users_custom_id

```
UNIQUE INDEX ON users(custom_id) WHERE custom_id IS NOT NULL
```

Purpose: Custom identity provider lookup.

#### IDX_user_sessions_expiry

```
INDEX ON user_sessions(expires_at) WHERE revoked = FALSE
```

Purpose: Session expiration cleanup daemon queries. Filters out already-revoked sessions.

#### IDX_user_sessions_user_id

```
INDEX ON user_sessions(user_id)
```

Purpose: FK cascade optimization and user session lookup.

---

### Multiplayer & Matchmaking

#### IDX_matchmaking_ticket_lookup

```
INDEX ON matchmaking_ticket(queue_name, status, created_at) WHERE status = 'queued'
```

Purpose: Matchmaker tick loop polling — fetches all active queued tickets within a queue, ordered by creation time. Partial index filters out non-queued tickets.

#### IDX_matchmaking_ticket_properties

```
GIN INDEX ON matchmaking_ticket USING gin(properties jsonb_path_ops)
```

Purpose: Custom property JSON matching during matchmaking evaluation. Optimized with jsonb_path_ops to reduce index size.

#### IDX_matchmaking_ticket_user_id

```
INDEX ON matchmaking_ticket(user_id)
```

Purpose: FK cascade optimization and per-user ticket lookup.

#### IDX_match_metadata_active_label

```
INDEX ON match_metadata(ended_at, authoritative) WHERE ended_at IS NULL
```

Purpose: Listing active matches with specific labels. Partial index filters out completed matches.

---

### Competitive

#### IDX_leaderboard_record_ranking_desc

```
INDEX ON leaderboard_record(leaderboard_id, score DESC, subscore DESC, update_time ASC) INCLUDE (expiry_time)
```

Purpose: Descending score ranking queries with covering filter columns for index-only scans.

#### IDX_leaderboard_record_ranking_asc

```
INDEX ON leaderboard_record(leaderboard_id, score ASC, subscore ASC, update_time ASC) INCLUDE (expiry_time)
```

Purpose: Ascending score ranking queries with covering filter columns for index-only scans.

#### IDX_leaderboard_record_owner_id

```
INDEX ON leaderboard_record(owner_id)
```

Purpose: FK cascade optimization and user lookup across leaderboards.

#### IDX_tournament_schedule

```
INDEX ON tournament(start_time, end_time)
```

Purpose: Scheduler engine queries for active and pending tournaments.

---

### Social

#### IDX_user_edge_source_lookup

```
INDEX ON user_edge(source_id, state, position DESC)
```

Purpose: Listing user friends filtered by state (friend/invite/blocked), sorted by position.

#### IDX_user_edge_dest_lookup

```
INDEX ON user_edge(destination_id, state)
```

Purpose: Checking reverse relationships (e.g., "has user B blocked user A?").

#### IDX_groups_name_active

```
UNIQUE INDEX ON groups(name) WHERE state = 0
```

Purpose: Enforce unique group names for active (non-deleted) groups.

#### IDX_group_edge_membership

```
INDEX ON group_edge(source_id, state, position DESC)
```

Purpose: Listing members of a group sorted by role and join date.

#### IDX_groups_category_search

```
INDEX ON groups(category, state, edge_count) WHERE state = 0
```

Purpose: Group search and filtering by category. Excludes soft-deleted groups.

---

### Communication

#### IDX_message_channel_history

```
INDEX ON message(channel_id, create_time DESC) WHERE deleted = FALSE
```

Purpose: Paginated channel message history (newest first). Partial index excludes soft-deleted messages.

#### IDX_channel_user_id_one

```
INDEX ON channel(user_id_one) WHERE user_id_one IS NOT NULL
```

Purpose: DM channel user lookup and FK cascade optimization.

#### IDX_channel_user_id_two

```
INDEX ON channel(user_id_two) WHERE user_id_two IS NOT NULL
```

Purpose: DM channel user lookup and FK cascade optimization.

#### IDX_message_sender

```
INDEX ON message(sender_id)
```

Purpose: Message sender lookup and FK cascade optimization.

#### IDX_notification_user_unread

```
INDEX ON notification(user_id, create_time DESC) WHERE delete_time IS NULL
```

Purpose: Pulling unread persistent notifications for a logged-in user.

#### IDX_notification_user_id

```
INDEX ON notification(user_id)
```

Purpose: FK cascade optimization and generic user notification queries.

#### IDX_notification_ttl

```
INDEX ON notification(delete_time) WHERE delete_time IS NOT NULL
```

Purpose: TTL expiration cleanup daemon queries.

---

### Data & Storage

#### IDX_storage_user_collection

```
INDEX ON storage(user_id, collection)
```

Purpose: Listing all objects in a collection for a specific user.

#### IDX_storage_public_read

```
INDEX ON storage(collection, key) WHERE permission_read = 2
```

Purpose: Public read lookup for shared/global objects.

#### IDX_storage_value

```
GIN INDEX ON storage USING gin(value jsonb_path_ops)
```

Purpose: Containment querying inside arbitrary JSONB user storage data. Optimized with jsonb_path_ops to reduce index size.

#### IDX_player_match_match_id

```
INDEX ON player_match(match_id)
```

Purpose: FK cascade optimization for match deletion.

---

### Economy

#### IDX_wallet_ledger_user_history

```
INDEX ON wallet_ledger(user_id, create_time DESC)
```

Purpose: Auditing wallet changesets, ordered newest first.

---

### Monetization

#### IDX_purchase_user_history

```
INDEX ON purchase(user_id, create_time DESC)
```

Purpose: Listing all purchases made by a specific player.

#### IDX_subscription_expiry

```
INDEX ON subscription(user_id, expiry_time DESC) WHERE active = TRUE
```

Purpose: Checking active subscription durations. Partial index filters inactive subscriptions.

#### IDX_subscription_user_id

```
INDEX ON subscription(user_id)
```

Purpose: FK cascade optimization and user subscription lookup.
