# ADR-0002: Use PostgreSQL as Primary Transactional Datastore

- **Status**: Accepted
- **Date**: 2026-07-02

---

## Context
Multiplayer games require reliable storage for persistent player profiles, ledger auditing, social graphs, and IAP transactions. These data models demand ACID compliance to prevent balance anomalies, duplication bugs, or inconsistent graphs.

At the same time, we need a datastore that supports JSON structures (for flexible NoSQL storage) and performs well under heavy read operations (such as leaderboards and active match indexing).

## Decision
We will use **PostgreSQL** as the primary transactional datastore.

PostgreSQL is selected because of:
1. **ACID Transactions**: Guarantees atomic writes for social edge inserts, economy mutations, and purchase records.
2. **JSONB Support**: Allows storage of arbitrary document-style data (user storage collections, wallets, matching ticket properties) with indexing (GIN).
3. **Covering Indexes**: Supports the `INCLUDE` clause, enabling high-performance Index-Only Scans for leaderboard rankings by keeping extra filtering columns directly in B-tree leaves without affecting sorting.

## Consequences
- We must enforce strict query optimization, routing reads to read replicas and writes to the primary.
- Foreign keys must be indexed to prevent sequential table scans on cascade deletes.
- Index partial filters must rely only on immutable functions (avoiding mutable calls like `NOW()`).
