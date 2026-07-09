# Entity Relationship Diagram (ERD)

> **Project:** Ultimate Game Engine — Multiplayer Game Server  
> **Last Updated:** 2026-07-08  
> **Status:** Draft

Table definitions and indexes live in [`database-design.md`](./database-design.md).

---

## Entity Relationships

### Identity & Access (TDD-01)

```
users
↓ 1:N
user_device
```

```
user_tombstone (standalone — soft delete record)
```

---



### Competitive (TDD-05, TDD-06)

```
leaderboard
↓ 1:N
leaderboard_record
↑ N:1
users
```



### Social — Friends (TDD-07)

```
users
↓ N:N (self-referencing, bidirectional edges)
user_edge
```

---

### Social — Guilds, Clans & Groups (TDD-09, TDD-16)

> Groups (TDD-16) reuse the `groups` and `group_edge` tables.
> The `groups.category` column differentiates guilds (category=0) from
> clubs, communities, and other group types.

```
users
↓ creator (1:N)
groups
↓ 1:N (bidirectional edges: group↔user)
group_edge
↑ N:1
users
```

---

### Communication — Chat (TDD-10)

```
message
↑ N:1 (sender)
users
```

---

### Communication — Notifications (TDD-11)

```
users
↓ 1:N (recipient)
notification
↑ N:1 (sender, optional)
users
```

---

### Data & Storage (TDD-12)

```
users
↓ 1:N
storage
```

---



### Economy (TDD-13)

> Wallet balance is stored as `users.wallet` (JSONB column).
> The ledger tracks all mutations for auditing.

```
users
↓ 1:N
wallet_ledger
```

---

### Monetization — IAP & Subscriptions (TDD-21)

```
users
↓ 1:N
purchase
```

```
users
↓ 1:N
subscription
```

---

### Console Admin (TDD-22)

```
console_user
↓ 1:N (logical connection; no foreign key constraint)
console_audit_log
```

```
console_acl_template (standalone template registry)
```

```
setting (standalone configuration store)
```

```
users
↓ 1:N
users_notes
```

---

### Infrastructure (TDD-20)

```
schema_version (standalone — tracks migration versions)
```
