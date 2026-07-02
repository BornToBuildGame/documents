# ADR-0004: Optimistic Concurrency Control (OCC) for Storage Writes

- **Status**: Accepted
- **Date**: 2026-07-02

---

## Context
The storage engine allows users and server tasks to save arbitrary JSON data (player inventories, quest state, configurations). In a multiplayer game, multiple client devices or background servers might attempt to write to the same user storage record concurrently.

If two writes occur in parallel without synchronization, the second write will overwrite the first one blindly (Lost Update problem). Pessimistic row locking (`SELECT FOR UPDATE`) prevents this but reduces throughput by forcing writes to queue, increasing the risk of database lock timeouts.

## Decision
We will use **Optimistic Concurrency Control (OCC)** for all storage engine document modifications.

Specifically:
1. Every storage record contains a `version` hash (MD5 hash of the data value).
2. When a client reads storage data, the server returns the value and the current `version`.
3. When the client writes back, it must supply the expected `version`.
4. The database executes the update with a conditional `WHERE version = expected_version`.
5. If the version matches, the write succeeds and a new hash is generated. If it fails (0 rows updated), the transaction aborts and returns `409 Conflict`.

Unconditional updates (upserts) are still supported if the version parameter is omitted by the caller.

## Consequences
- Clients must handle `409 Conflict` errors by fetching the latest state, merging changes, and re-attempting the write.
- Allows high write throughput without table row locks, scaling well for concurrent player activities.
