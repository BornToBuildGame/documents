# Database Design

> **Project:** Ultimate Game Engine — Multiplayer Game Server  
> **Last Updated:** 2026-07-08  
> **Status:** Draft

Entity relationships live in [`ERD.md`](./ERD.md).

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

#### user_device

Source: [TDD-01](../TDD/01_user_authentication.md)

```
id              VARCHAR(128)    PRIMARY KEY
user_id         UUID            NOT NULL REFERENCES users(id) ON DELETE CASCADE
create_time     TIMESTAMPTZ     DEFAULT CURRENT_TIMESTAMP NOT NULL
update_time     TIMESTAMPTZ     DEFAULT CURRENT_TIMESTAMP NOT NULL
```

#### user_tombstone

Source: [TDD-01](../TDD/01_user_authentication.md)

```
user_id         UUID            PRIMARY KEY
create_time     TIMESTAMPTZ     DEFAULT CURRENT_TIMESTAMP NOT NULL
```

---


### Competitive

#### leaderboard

Source: [TDD-05](../TDD/05_leaderboards.md)

```
id              VARCHAR(128)    PRIMARY KEY
authoritative   BOOLEAN         DEFAULT FALSE NOT NULL
sort_order      SMALLINT        DEFAULT 1 NOT NULL                  -- 0=ascending, 1=descending
operator        SMALLINT        DEFAULT 0 NOT NULL                  -- 0=best, 1=set, 2=increment, 3=decrement
reset_schedule  VARCHAR(64)                                         -- Cron expression
metadata        JSONB           DEFAULT '{}'::jsonb NOT NULL
create_time     TIMESTAMPTZ     DEFAULT CURRENT_TIMESTAMP NOT NULL
category        INT             DEFAULT 0 NOT NULL
description     VARCHAR(255)    DEFAULT '' NOT NULL
duration        INT             DEFAULT 0 NOT NULL
end_time        TIMESTAMPTZ     DEFAULT '1970-01-01 00:00:00 UTC' NOT NULL
join_required   BOOLEAN         DEFAULT FALSE NOT NULL
max_size        INT             DEFAULT 100000000 NOT NULL
max_num_score   INT             DEFAULT 100000000 NOT NULL
title           VARCHAR(255)    DEFAULT '' NOT NULL
size            INT             DEFAULT 0 NOT NULL
start_time      TIMESTAMPTZ     DEFAULT CURRENT_TIMESTAMP NOT NULL
```

#### leaderboard_record

Source: [TDD-05](../TDD/05_leaderboards.md)

```
leaderboard_id  VARCHAR(128)    NOT NULL REFERENCES leaderboard(id) ON DELETE CASCADE
expiry_time     TIMESTAMPTZ     NOT NULL DEFAULT '1970-01-01 00:00:00+00'
score           BIGINT          DEFAULT 0 NOT NULL                  -- CHECK (score >= 0)
subscore        BIGINT          DEFAULT 0 NOT NULL                  -- CHECK (subscore >= 0)
owner_id        UUID            NOT NULL REFERENCES users(id) ON DELETE CASCADE
username        VARCHAR(128)
num_score       INT             DEFAULT 1 NOT NULL                  -- Number of submissions, CHECK (num_score >= 0)
max_num_score   INT             DEFAULT 0 NOT NULL                  -- Max submissions limit, 0=unlimited, CHECK (max_num_score >= 0)
metadata        JSONB           DEFAULT '{}'::jsonb NOT NULL
create_time     TIMESTAMPTZ     DEFAULT now() NOT NULL
update_time     TIMESTAMPTZ     DEFAULT now() NOT NULL
PRIMARY KEY (leaderboard_id, expiry_time, score, subscore, owner_id)
```

Notes:
- Also used by the Tournament system (TDD-06). Tournament scores are scoped via `expiry_time`.
- Performance: Ultimate Game Engine maintains an in-memory rank cache to resolve rankings dynamically, bypassing expensive DB queries.



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

### Console Admin

#### console_user

Source: [TDD-22](../TDD/22_console_admin.md)

```
id              UUID            PRIMARY KEY DEFAULT gen_random_uuid()
username        VARCHAR(128)    UNIQUE NOT NULL
email           VARCHAR(255)    UNIQUE NOT NULL
password        BYTEA
role            SMALLINT        DEFAULT 1 NOT NULL                  -- 1=admin, 2=developer, 3=maintainer, 4=readonly
create_time     TIMESTAMPTZ     DEFAULT CURRENT_TIMESTAMP NOT NULL
update_time     TIMESTAMPTZ     DEFAULT CURRENT_TIMESTAMP NOT NULL
```

#### console_audit_log

Source: [TDD-22](../TDD/22_console_admin.md)

```
id              UUID            PRIMARY KEY DEFAULT gen_random_uuid()
user_id         UUID            NOT NULL REFERENCES console_user(id) ON DELETE CASCADE
action          VARCHAR(128)    NOT NULL
resource        VARCHAR(128)    NOT NULL
payload         JSONB           DEFAULT '{}'::jsonb NOT NULL
create_time     TIMESTAMPTZ     DEFAULT CURRENT_TIMESTAMP NOT NULL
```

#### console_acl_template

Source: [TDD-22](../TDD/22_console_admin.md)

```
role            SMALLINT        PRIMARY KEY
permissions     JSONB           DEFAULT '{}'::jsonb NOT NULL
```

#### setting

Source: [TDD-22](../TDD/22_console_admin.md)

```
key             VARCHAR(128)    PRIMARY KEY
value           TEXT            NOT NULL
```

#### users_notes

Source: [TDD-22](../TDD/22_console_admin.md)

```
user_id         UUID            PRIMARY KEY REFERENCES users(id) ON DELETE CASCADE
note            TEXT            NOT NULL
create_time     TIMESTAMPTZ     DEFAULT CURRENT_TIMESTAMP NOT NULL
update_time     TIMESTAMPTZ     DEFAULT CURRENT_TIMESTAMP NOT NULL
```

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

---

### Console Admin

#### IDX_console_audit_log_user

```
INDEX ON console_audit_log(user_id, create_time DESC)
```

Purpose: Listing audit logs for a specific console user.

#### IDX_users_notes_user_id

```
INDEX ON users_notes(user_id)
```

Purpose: Quick lookup of notes associated with a player.

