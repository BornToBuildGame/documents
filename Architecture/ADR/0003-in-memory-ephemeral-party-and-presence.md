# ADR-0003: In-Memory Ephemeral Party and Presence Storage

- **Status**: Accepted
- **Date**: 2026-07-02

---

## Context
Real-time features like active online presence status, matchmaking queues, and group party chats are highly volatile. A single status update or player movement event happens multiple times per second. Saving these events to a persistent database like PostgreSQL would result in massive write amplification, locking bottlenecks, and disk exhaustion.

## Decision
We will handle all online presence states, party sessions, and matchmaking ticket pools **entirely in-memory** or distributed across nodes using **Redis Cluster**.

Specifically:
1. **No DB writes**: Active parties and online presences do not write to PostgreSQL.
2. **Cluster Communication**: Multi-node cluster nodes exchange presence changes and party updates using Redis Pub/Sub channels.
3. **Session Eviction**: If a player disconnects, they are kept in a disconnected state for a 30-second grace window. If the window expires without reconnection, they are pruned from the in-memory registry.

## Consequences
- If the server node crashes or the Redis cluster restarts, active party sessions and online status states are lost, and clients must re-establish sessions.
- This is acceptable since parties and online presence are ephemeral client sessions, not long-term progression.
- Matches are checkpointed to PostgreSQL (see ADR-0004/TDD-15) for crash recovery, but the raw live connection presence is ephemeral.
