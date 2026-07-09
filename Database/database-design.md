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
id                        UUID            PRIMARY KEY
username                  VARCHAR(128)    UNIQUE NOT NULL
display_name              VARCHAR(255)
avatar_url                VARCHAR(512)
lang_tag                  VARCHAR(18)     DEFAULT 'en' NOT NULL
location                  VARCHAR(255)
timezone                  VARCHAR(255)
metadata                  JSONB           DEFAULT '{}'::jsonb NOT NULL
wallet                    JSONB           DEFAULT '{}'::jsonb NOT NULL
email                     VARCHAR(255)    UNIQUE
password                  BYTEA           CHECK (length(password) < 32000)
facebook_id               VARCHAR(128)    UNIQUE
google_id                 VARCHAR(128)    UNIQUE
gamecenter_id             VARCHAR(128)    UNIQUE
steam_id                  VARCHAR(128)    UNIQUE
custom_id                 VARCHAR(128)    UNIQUE
apple_id                  VARCHAR(128)    UNIQUE
facebook_instant_game_id  VARCHAR(128)    UNIQUE
edge_count                INT             DEFAULT 0 NOT NULL CHECK (edge_count >= 0)
create_time               TIMESTAMPTZ     DEFAULT now() NOT NULL
update_time               TIMESTAMPTZ     DEFAULT now() NOT NULL
verify_time               TIMESTAMPTZ     DEFAULT '1970-01-01 00:00:00 UTC' NOT NULL
disable_time              TIMESTAMPTZ     DEFAULT '1970-01-01 00:00:00 UTC' NOT NULL
```

Notes:
- `wallet` stores virtual currency balances as a JSONB map (e.g., `{"coins": 500, "gems": 10}`). See [TDD-13](../TDD/13_economy_system.md) for economy system details.
- Social provider ID columns (`apple_id`, `google_id`, etc.) support account linking across multiple identity providers.
- A default system user record with UUID `00000000-0000-0000-0000-000000000000` must be inserted during database initialization. This system account owns global storage objects and acts as the destination fallback for orphaned purchase/subscription records on user deletion.

#### user_device

Source: [TDD-01](../TDD/01_user_authentication.md)

```
id                  VARCHAR(128)    PRIMARY KEY
user_id             UUID            NOT NULL REFERENCES users(id) ON DELETE CASCADE
preferences         JSONB           DEFAULT '{}'::jsonb NOT NULL
push_token_amazon   VARCHAR(512)    DEFAULT '' NOT NULL
push_token_android  VARCHAR(512)    DEFAULT '' NOT NULL
push_token_huawei   VARCHAR(512)    DEFAULT '' NOT NULL
push_token_ios      VARCHAR(512)    DEFAULT '' NOT NULL
push_token_web      VARCHAR(512)    DEFAULT '' NOT NULL
UNIQUE (user_id, id)
```

#### user_tombstone

Source: [TDD-01](../TDD/01_user_authentication.md)

```
user_id             UUID            NOT NULL UNIQUE
create_time         TIMESTAMPTZ     DEFAULT now() NOT NULL
PRIMARY KEY (create_time, user_id)
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
create_time     TIMESTAMPTZ     DEFAULT now() NOT NULL
category        SMALLINT        DEFAULT 0 NOT NULL CHECK (category >= 0)
description     VARCHAR(255)    DEFAULT '' NOT NULL
duration        INT             DEFAULT 0 NOT NULL CHECK (duration >= 0)
end_time        TIMESTAMPTZ     DEFAULT '1970-01-01 00:00:00 UTC' NOT NULL
join_required   BOOLEAN         DEFAULT FALSE NOT NULL
max_size        INT             DEFAULT 100000000 NOT NULL CHECK (max_size > 0)
max_num_score   INT             DEFAULT 1000000 NOT NULL CHECK (max_num_score > 0)
title           VARCHAR(255)    DEFAULT '' NOT NULL
size            INT             DEFAULT 0 NOT NULL
start_time      TIMESTAMPTZ     DEFAULT now() NOT NULL
enable_ranks    BOOLEAN         DEFAULT true NOT NULL
```

#### leaderboard_record

Source: [TDD-05](../TDD/05_leaderboards.md)

```
leaderboard_id  VARCHAR(128)    NOT NULL REFERENCES leaderboard(id) ON DELETE CASCADE
expiry_time     TIMESTAMPTZ     NOT NULL DEFAULT '1970-01-01 00:00:00 UTC'
score           BIGINT          DEFAULT 0 NOT NULL CHECK (score >= 0)
subscore        BIGINT          DEFAULT 0 NOT NULL CHECK (subscore >= 0)
owner_id        UUID            NOT NULL
username        VARCHAR(128)
num_score       INT             DEFAULT 1 NOT NULL CHECK (num_score >= 0)
max_num_score   INT             DEFAULT 1000000 NOT NULL CHECK (max_num_score > 0)
metadata        JSONB           DEFAULT '{}'::jsonb NOT NULL
create_time     TIMESTAMPTZ     DEFAULT now() NOT NULL
update_time     TIMESTAMPTZ     DEFAULT now() NOT NULL
PRIMARY KEY (leaderboard_id, expiry_time, score, subscore, owner_id)
UNIQUE (owner_id, leaderboard_id, expiry_time)
```

Notes:
- Also used by the Tournament system (TDD-06). Tournament scores are scoped via `expiry_time`.
- Performance: Ultimate Game Engine maintains an in-memory rank cache to resolve rankings dynamically, bypassing expensive DB queries.



### Social

#### user_edge

Source: [TDD-07](../TDD/07_friends_system.md)

```
source_id       UUID            NOT NULL REFERENCES users(id) ON DELETE CASCADE CHECK (source_id <> '00000000-0000-0000-0000-000000000000')
position        BIGINT          NOT NULL
update_time     TIMESTAMPTZ     DEFAULT now() NOT NULL
destination_id  UUID            NOT NULL REFERENCES users(id) ON DELETE CASCADE CHECK (destination_id <> '00000000-0000-0000-0000-000000000000')
state           SMALLINT        DEFAULT 0 NOT NULL                  -- friend(0), invite_sent(1), invite_received(2), blocked(3)
metadata        JSONB           DEFAULT '{}'::jsonb NOT NULL
PRIMARY KEY (source_id, state, position)
UNIQUE (source_id, destination_id)
```

Notes:
- Bidirectional directed adjacency list. Each friend relationship consists of two rows (A→B and B→A).
- `position` is an epoch-millisecond value used for sorting.

#### groups

Source: [TDD-09](../TDD/09_guilds_clans.md), [TDD-16](../TDD/16_groups.md)

```
id              UUID            UNIQUE NOT NULL
creator_id      UUID            NOT NULL
name            VARCHAR(255)    UNIQUE NOT NULL
description     VARCHAR(255)
avatar_url      VARCHAR(512)
lang_tag        VARCHAR(18)     DEFAULT 'en' NOT NULL
metadata        JSONB           DEFAULT '{}'::jsonb NOT NULL
state           SMALLINT        DEFAULT 0 NOT NULL CHECK (state >= 0)  -- open(0), closed(1)
edge_count      INT             DEFAULT 0 NOT NULL CHECK (edge_count >= 1 AND edge_count <= max_count)
max_count       INT             DEFAULT 100 NOT NULL CHECK (max_count >= 1)
create_time     TIMESTAMPTZ     DEFAULT now() NOT NULL
update_time     TIMESTAMPTZ     DEFAULT now() NOT NULL
disable_time    TIMESTAMPTZ     DEFAULT '1970-01-01 00:00:00 UTC' NOT NULL
PRIMARY KEY (disable_time, lang_tag, edge_count, id)
```

Notes:
- `edge_count` tracks the current number of members. Reconciled periodically by a background job.
- Groups/guilds are soft-disabled by setting `disable_time`.

#### group_edge

Source: [TDD-09](../TDD/09_guilds_clans.md), [TDD-16](../TDD/16_groups.md)

```
source_id       UUID            NOT NULL CHECK (source_id <> '00000000-0000-0000-0000-000000000000')
position        BIGINT          NOT NULL
update_time     TIMESTAMPTZ     DEFAULT now() NOT NULL
destination_id  UUID            NOT NULL CHECK (destination_id <> '00000000-0000-0000-0000-000000000000')
state           SMALLINT        DEFAULT 0 NOT NULL                  -- superadmin(0), admin(1), member(2), join_request(3), banned(4)
PRIMARY KEY (source_id, state, position)
UNIQUE (source_id, destination_id)
```

Notes:
- Bidirectional edges: Group→User and User→Group pairs.
- Role hierarchy: superadmin (0) > admin (1) > member (2) > join_request (3) > banned (4).
- Polymorphic edges: `source_id` and `destination_id` lack FK constraints. Deleting users/groups requires application-level cleanup.

---

### Communication

#### message

Source: [TDD-10](../TDD/10_chat_system.md)

```
id                UUID            UNIQUE NOT NULL
code              SMALLINT        DEFAULT 0 NOT NULL                  -- chat(0), chat_update(1), chat_remove(2), group_join(3)...
sender_id         UUID            NOT NULL REFERENCES users(id) ON DELETE CASCADE
username          VARCHAR(128)    NOT NULL
stream_mode       SMALLINT        NOT NULL
stream_subject    UUID            NOT NULL
stream_descriptor UUID            NOT NULL
stream_label      VARCHAR(128)    NOT NULL
content           JSONB           DEFAULT '{}'::jsonb NOT NULL
create_time       TIMESTAMPTZ     DEFAULT now() NOT NULL
update_time       TIMESTAMPTZ     DEFAULT now() NOT NULL
PRIMARY KEY (stream_mode, stream_subject, stream_descriptor, stream_label, create_time, id)
UNIQUE (sender_id, id)
```

Notes:
- Stream-based chat system. Channels are virtual streams defined by `stream_mode`, `stream_subject`, `stream_descriptor`, and `stream_label`.
- 90-day retention policy; a background daemon purges older messages.

#### notification

Source: [TDD-11](../TDD/11_notifications.md)

```
id              UUID            UNIQUE NOT NULL
user_id         UUID            NOT NULL REFERENCES users(id) ON DELETE CASCADE
subject         VARCHAR(255)    NOT NULL
content         JSONB           DEFAULT '{}'::jsonb NOT NULL
code            SMALLINT        NOT NULL                            -- Negative values are system reserved
sender_id       UUID            NOT NULL
create_time     TIMESTAMPTZ     DEFAULT now() NOT NULL
PRIMARY KEY (user_id, create_time, id)
```

Notes:
- The primary key is structured as a composite key `(user_id, create_time, id)` to optimize index scanning by user and date, particularly on clustered databases like CockroachDB.
- Max 1,000 persistent notifications per user; oldest are pruned when the cap is exceeded.

---

### Data & Storage

#### storage

Source: [TDD-12](../TDD/12_storage_engine.md)

```
collection  VARCHAR(128)    NOT NULL
key         VARCHAR(128)    NOT NULL
user_id     UUID            NOT NULL REFERENCES users(id) ON DELETE CASCADE
value       JSONB           DEFAULT '{}'::jsonb NOT NULL
version     VARCHAR(32)     NOT NULL                            -- MD5 hash string of value object
read        SMALLINT        DEFAULT 1 NOT NULL CHECK (read >= 0)
write       SMALLINT        DEFAULT 1 NOT NULL CHECK (write >= 0)
create_time TIMESTAMPTZ     DEFAULT now() NOT NULL
update_time TIMESTAMPTZ     DEFAULT now() NOT NULL
PRIMARY KEY (collection, key, user_id)
```

Notes:
- NoSQL-style document storage backed by PostgreSQL JSONB.
- Optimistic concurrency control via `version` (MD5 hash of the value).
- `user_id = '00000000-0000-0000-0000-000000000000'` denotes global (server-owned) objects. The system user record must be seeded in the `users` table to satisfy the foreign key reference constraint.

### Economy

#### wallet_ledger

Source: [TDD-13](../TDD/13_economy_system.md)

```
id          UUID            UNIQUE NOT NULL
user_id     UUID            NOT NULL REFERENCES users(id) ON DELETE CASCADE
changeset   JSONB           NOT NULL                            -- e.g., {"coins": 100} or {"gems": -10}
metadata    JSONB           DEFAULT '{}'::jsonb NOT NULL        -- e.g., {"reason": "quest_reward"}
create_time TIMESTAMPTZ     DEFAULT now() NOT NULL
update_time TIMESTAMPTZ     DEFAULT now() NOT NULL
PRIMARY KEY (user_id, create_time, id)
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
transaction_id  VARCHAR(512)    PRIMARY KEY CHECK (length(transaction_id) > 0)
user_id         UUID            DEFAULT '00000000-0000-0000-0000-000000000000' NOT NULL REFERENCES users(id) ON DELETE SET DEFAULT
product_id      VARCHAR(512)    NOT NULL
store           SMALLINT        DEFAULT 0 NOT NULL                  -- AppleAppStore(0), GooglePlay(1), Huawei(2)
raw_response    JSONB           DEFAULT '{}'::jsonb NOT NULL
purchase_time   TIMESTAMPTZ     DEFAULT now() NOT NULL
refund_time     TIMESTAMPTZ     DEFAULT '1970-01-01 00:00:00 UTC' NOT NULL
environment     SMALLINT        DEFAULT 0 NOT NULL                  -- Unknown(0), Sandbox(1), Production(2)
create_time     TIMESTAMPTZ     DEFAULT now() NOT NULL
update_time     TIMESTAMPTZ     DEFAULT now() NOT NULL
```

Notes:
- `user_id` defaults to the system user UUID on delete (`ON DELETE SET DEFAULT`) to preserve analytical revenue records.
- `refund_time` defaults to 1970 and is set to the actual refund timestamp when a chargeback/refund notification is received.

#### subscription

Source: [TDD-21](../TDD/21_iap_validation.md)

```
original_transaction_id VARCHAR(512)    PRIMARY KEY CHECK (length(original_transaction_id) > 0)
user_id                 UUID            DEFAULT '00000000-0000-0000-0000-000000000000' NOT NULL REFERENCES users(id) ON DELETE SET DEFAULT
product_id              VARCHAR(512)    NOT NULL
store                   SMALLINT        DEFAULT 0 NOT NULL              -- AppleAppStore(0), GooglePlay(1), Huawei(2)
raw_response            JSONB           DEFAULT '{}'::jsonb NOT NULL
raw_notification        JSONB           DEFAULT '{}'::jsonb NOT NULL
purchase_time           TIMESTAMPTZ     DEFAULT now() NOT NULL
expire_time             TIMESTAMPTZ     NOT NULL
refund_time             TIMESTAMPTZ     DEFAULT '1970-01-01 00:00:00 UTC' NOT NULL
environment             SMALLINT        DEFAULT 0 NOT NULL              -- Unknown(0), Sandbox(1), Production(2)
create_time             TIMESTAMPTZ     DEFAULT now() NOT NULL
update_time             TIMESTAMPTZ     DEFAULT now() NOT NULL
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
id                  UUID            PRIMARY KEY
username            VARCHAR(128)    UNIQUE NOT NULL
email               VARCHAR(255)    UNIQUE NOT NULL
password            BYTEA           CHECK (length(password) < 32000)
acl                 JSONB           DEFAULT '{"admin":false}'::jsonb NOT NULL
metadata            JSONB           DEFAULT '{}'::jsonb NOT NULL
create_time         TIMESTAMPTZ     DEFAULT now() NOT NULL
update_time         TIMESTAMPTZ     DEFAULT now() NOT NULL
disable_time        TIMESTAMPTZ     DEFAULT '1970-01-01 00:00:00 UTC' NOT NULL
mfa_secret          BYTEA           DEFAULT NULL
mfa_recovery_codes  BYTEA           DEFAULT NULL
mfa_required        BOOLEAN         DEFAULT FALSE NOT NULL
```

#### console_audit_log

Source: [TDD-22](../TDD/22_console_admin.md)

```
id                  UUID            UNIQUE NOT NULL
console_user_id     UUID            NOT NULL
console_username    TEXT            NOT NULL
email               TEXT            NOT NULL
action              TEXT            NOT NULL
resource            TEXT            NOT NULL
message             TEXT            NOT NULL
metadata            JSONB           DEFAULT '{}'::jsonb NOT NULL
create_time         TIMESTAMPTZ     DEFAULT now() NOT NULL
PRIMARY KEY (create_time, console_username, action, resource, id)
```

#### console_acl_template

Source: [TDD-22](../TDD/22_console_admin.md)

```
id                  UUID            PRIMARY KEY
name                VARCHAR(64)     UNIQUE NOT NULL CHECK (length(name) > 0)
description         VARCHAR(64)     DEFAULT '' NOT NULL
acl                 JSONB           DEFAULT '{}'::jsonb NOT NULL
create_time         TIMESTAMPTZ     DEFAULT now() NOT NULL
update_time         TIMESTAMPTZ     DEFAULT now() NOT NULL
```

#### setting

Source: [TDD-22](../TDD/22_console_admin.md)

```
name                VARCHAR(64)     PRIMARY KEY CHECK (length(name) > 0) CONSTRAINT setting_name_uniq UNIQUE
value               JSONB           DEFAULT '{}'::jsonb NOT NULL
update_time         TIMESTAMPTZ     DEFAULT now() NOT NULL
```

#### users_notes

Source: [TDD-22](../TDD/22_console_admin.md)

```
id                  UUID            UNIQUE NOT NULL
user_id             UUID            NOT NULL REFERENCES users(id) ON DELETE CASCADE
create_time         TIMESTAMPTZ     DEFAULT now() NOT NULL
update_time         TIMESTAMPTZ     DEFAULT now() NOT NULL
note                TEXT            NOT NULL
create_id           UUID            DEFAULT NULL
update_id           UUID            DEFAULT NULL
PRIMARY KEY (user_id, create_time, id)
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

#### IDX_users_gamecenter_id

```
UNIQUE INDEX ON users(gamecenter_id) WHERE gamecenter_id IS NOT NULL
```

Purpose: Game Center identity provider lookup.

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

#### IDX_users_facebook_instant_game_id

```
UNIQUE INDEX ON users(facebook_instant_game_id) WHERE facebook_instant_game_id IS NOT NULL
```

Purpose: Facebook Instant Games identity provider lookup.

#### IDX_users_display_name_trgm

```
INDEX ON users USING GIN (display_name gin_trgm_ops)
```

Purpose: Full-text trigram index for player search by display name.

#### IDX_user_device_user_id

```
INDEX ON user_device(user_id)
```

Purpose: FK cascade optimization.

---

### Competitive

#### IDX_leaderboard_create_time_id

```
INDEX ON leaderboard (create_time ASC, id ASC)
```

Purpose: Sorting leaderboards by creation time.

#### IDX_leaderboard_duration_start_time_end_time_category

```
INDEX ON leaderboard (duration, start_time, end_time DESC, category)
```

Purpose: Querying scheduled tournaments/leaderboards filtered by duration, start, end time, and category.

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

#### IDX_leaderboard_record_owner_id_expiry_time_leaderboard_id

```
INDEX ON leaderboard_record (owner_id, expiry_time, leaderboard_id)
```

Purpose: Querying leaderboard records for a specific owner across active/expired bounds.

---

### Social

#### IDX_user_edge_fk_destination_id

```
INDEX ON user_edge (destination_id)
```

Purpose: Optimizing FK lookups on destination user deletes.

#### IDX_groups_edge_count_update_time_id

```
INDEX ON groups (disable_time, edge_count, update_time, id)
```

Purpose: Querying groups by membership count and update time.

#### IDX_groups_update_time_edge_count_id

```
INDEX ON groups (disable_time, update_time, edge_count, id)
```

Purpose: Querying groups by update time and membership count.

---

### Communication

#### IDX_message_sender

```
INDEX ON message(sender_id)
```

Purpose: Message sender lookup and FK cascade optimization.

#### IDX_notification_user_id

```
INDEX ON notification(user_id, create_time DESC)
```

Purpose: Pulling notifications for a user sorted by creation time.

---

### Data & Storage

#### IDX_storage_collection_read_user_id_key

```
INDEX ON storage (collection, read, user_id, key)
```

Purpose: Querying collections with read permissions for a user.

#### IDX_storage_collection_read_key_user_id

```
INDEX ON storage (collection, read, key, user_id)
```

Purpose: Querying collections with read permissions by key.

#### IDX_storage_collection_user_id_read_key

```
INDEX ON storage (collection, user_id, read, key)
```

Purpose: Querying storage by user collection and read permissions.

#### IDX_storage_auto_index_fk_user_id

```
INDEX ON storage (user_id)
```

Purpose: FK cascade optimization.

#### IDX_storage_value_gin

```
GIN INDEX ON storage USING gin(value jsonb_path_ops)
```

Purpose: Containment querying inside arbitrary JSONB user storage data. Optimized with jsonb_path_ops to reduce index size.

---

### Economy

#### IDX_wallet_ledger_user_history

```
INDEX ON wallet_ledger(user_id, create_time DESC)
```

Purpose: Auditing wallet changesets, ordered newest first.

---

### Monetization

#### IDX_purchase_time_user_id_transaction_id

```
INDEX ON purchase (purchase_time DESC, user_id DESC, transaction_id DESC)
```

Purpose: Listing all purchases made by users sorted by purchase time.

#### IDX_subscription_time_user_id_transaction_id

```
INDEX ON subscription (purchase_time DESC, user_id DESC, original_transaction_id DESC)
```

Purpose: Checking active subscription durations and transaction history.

---

### Console Admin

#### IDX_users_notes_user_id

```
INDEX ON users_notes(user_id)
```

Purpose: Quick lookup of notes associated with a player.


