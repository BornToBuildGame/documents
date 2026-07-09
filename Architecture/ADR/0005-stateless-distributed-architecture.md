# 5. Stateless Distributed Architecture

Date: 2026-07-08

## Status

Accepted

## Context

The initial architectural design included explicit database-backed persistence for ephemeral state, including user sessions, matchmaking tickets, match states, and active authoritative matches. This approach introduces significant database overhead, high read/write contention, and MVCC bloat in PostgreSQL, particularly for highly concurrent realtime game features. 

To support the scalability targets required by a multi-node, horizontally scaled game backend (aligned with the Ultimate Game Engine reference architecture), the server needs to minimize its reliance on persistent storage for transient game data.

## Decision

We will shift to a **Stateless Distributed Architecture** for transient features:

1. **Session Management**: Migrate from database-backed `user_sessions` to stateless JWT (JSON Web Tokens) validated cryptographically by any node.
2. **Matchmaking & Matches**: Drop `matchmaking_ticket`, `matchmaker_match`, `match_state`, `match_history`, and `player_match` tables. 
   - Matchmaker state will be distributed in-memory using an in-memory mesh (e.g., Redis or a native clustering protocol).
   - Authoritative match state runs entirely in-memory on the assigned node.
   - A distributed registry will map `match_id` to the specific node processing the match loop.
3. **Inter-Node Routing**: Client inputs reaching any node via WebSocket will be routed to the correct authoritative match node using an internal pub/sub or gRPC mesh.
4. **Tournaments**: Instead of maintaining a separate `tournament` table, tournaments will be implemented directly on top of the `leaderboard` table using duration and scheduling fields.

## Consequences

- **Positive**: Substantially reduces PostgreSQL write pressure and eliminates MVCC bloat for high-frequency operations.
- **Positive**: Enables linear horizontal scaling of server nodes by decoupling compute from persistent storage.
- **Positive**: Aligns closely with the Ultimate Game Engine reference architecture.
- **Negative**: Adds complexity to the deployment infrastructure, requiring a distributed caching/pub-sub layer (e.g., Redis or clustered membership) for inter-node communication.
- **Negative**: Match states are ephemeral; if a node crashes, running matches on that node are lost unless custom storage persists them periodically.
