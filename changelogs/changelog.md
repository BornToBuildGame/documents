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
