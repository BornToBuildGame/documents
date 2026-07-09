**
2026-07-09
Synced database design, ERD, TDDs, and PRD with reference engine migrations (19 SQL scripts).
Reason: Resolved mismatches, missing columns/indexes, incorrect PK/FK strategies, and updated Console Admin ACL and messaging stream architectures.
**

**
2026-07-08
Migrated to stateless distributed architecture to align with reference engine (ADR-0005). Removed user_sessions, matchmaking, and match tables. Added user_device, user_tombstone, and console admin tables. Merged tournament into leaderboard.
Reason: to improve database performance, reduce MVCC bloat, and support horizontal scaling.
**

**
2026-07-08
Aligned leaderboard database schema and architecture with Ultimate Game Engine implementation (05_leaderboards.md, database-design.md)
Reason: to support Ultimate Game Engine's native in-memory caching and optimized querying instead of custom SQL CTE queries, matching their primary key structures.
**

**
2026-07-08
Optimized database schema for performance and integrity (database-design.md)
Reason: Addressed risks identified in design review: removed users.edge_count to prevent MVCC bloat, optimized GIN indexes with jsonb_path_ops, documented polymorphic edge cleanup strategies, and dropped redundant idx_player_match_history_lookup index.
**

**
2026-07-08
Initial database design documents created (ERD.md, database-design.md)
Reason: Designed complete PostgreSQL schema from TDD-01 through TDD-21 — 2 enum types, 17 persisted tables, 37 indexes across 9 feature domains (Identity, Multiplayer, Competitive, Social, Communication, Data, Economy, Monetization, Infrastructure)
**
