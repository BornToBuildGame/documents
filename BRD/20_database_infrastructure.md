# BRD-20: Database & Infrastructure

> **Project:** Ultimate Game Engine — Multiplayer Game Server  
> **Feature:** Database & Infrastructure  
> **Version:** 1.0  
> **Last Updated:** 2026-07-01  
> **Status:** Draft  
> **Priority:** Must Have
---
## 1. Purpose

Define the requirements for the database layer, server infrastructure, deployment, scaling, monitoring, and operational tooling. PostgreSQL serves as the primary database, with additional infrastructure supporting high availability, observability, and enterprise-grade operations.
---
## 2. Scope

### In Scope
- PostgreSQL database requirements
- Database schema management and migrations
- Connection pooling
- Backup and recovery
- Server deployment configuration
- Horizontal scaling
- Monitoring and observability
- Logging infrastructure
- Configuration management
- Admin console

### Out of Scope
- Cloud provider-specific services (AWS, GCP, Azure specifics)
- CDN configuration
- Client-side infrastructure
- CI/CD pipeline details
---
## 3. Stakeholders

| Role | Interest |
|------|----------|
| Server Administrators | Reliable operations, monitoring, backups |
| DevOps Engineers | Deployment, scaling, infrastructure as code |
| Database Administrators | Schema management, performance, backup |
| Game Developers | Reliable infrastructure, development tooling |
---
## 4. Functional Requirements

### FR-DB-001: PostgreSQL Database
- **Priority:** Must
- PostgreSQL shall be the primary persistent data store.
- Minimum version: PostgreSQL 12+.
- Stores:

| Data | Table(s) |
|------|----------|
| Users and accounts | `users` |
| Friend relationships | `user_edge` |
| Groups / Guilds | `groups`, `group_edge` |
| Storage objects | `storage` |
| Leaderboard config | `leaderboard` |
| Leaderboard records | `leaderboard_record` |
| Chat messages | `message` |
| Notifications | `notification` |
| Wallet ledger | `wallet_ledger` |
| Purchase receipts | `purchase` |

### FR-DB-002: Schema Migrations
- **Priority:** Must
- The server shall manage database schema via versioned migrations.
- Migrations run automatically on server startup.
- Support for:
  - Forward migration (upgrade)
  - Schema version tracking
  - Idempotent migrations (safe to re-run)
- Schema version table tracks current migration state.

### FR-DB-003: Connection Pooling
- **Priority:** Must
- The server shall use connection pooling for database access.
- Configurable pool parameters:
  - `max_open_conns`: maximum open connections (default: 100)
  - `max_idle_conns`: maximum idle connections (default: 100)
  - `conn_max_lifetime`: maximum connection lifetime
  - `conn_max_idle_time`: maximum idle time before close
- Support for external connection poolers (PgBouncer).

### FR-DB-004: Backup and Recovery
- **Priority:** Must
- Support for PostgreSQL backup strategies:
  - `pg_dump` for logical backups
  - WAL archiving for point-in-time recovery
  - Streaming replication for hot standby
- Configurable backup schedules.
- Recovery procedures documented.
- Backup verification testing.

### FR-DB-005: Server Configuration
- **Priority:** Must
- Server shall be configurable via:
  - Configuration file (YAML)
  - Environment variables
  - Command-line flags
- Configuration categories:

| Category | Parameters |
|----------|-----------|
| **Server** | name, port, data_dir |
| **Database** | address, port, database, user, password, pool settings |
| **Runtime** | lua_path, js_path, go_path |
| **Session** | token_expiry, refresh_expiry, encryption_key |
| **Socket** | max_message_size, max_request_size, read_buffer, write_buffer |
| **Matchmaker** | tick_rate, max_tickets, interval |
| **Leaderboard** | callback_queue_size |
| **Logger** | level, format, stdout, rotation |
| **Metrics** | namespace, prometheus_port |
| **TLS** | cert_file, key_file |

### FR-DB-006: Horizontal Scaling
- **Priority:** Should
- The server shall support running multiple nodes.
- Node discovery: configurable (static list, DNS, etcd, Consul).
- Data that must be shared across nodes:
  - Database (PostgreSQL — shared)
  - Presence (distributed via gossip or message bus)
  - Matchmaker state (centralized or distributed)
  - Match routing (redirect to correct node)
- Stateless nodes for REST/gRPC (behind load balancer).
- Sticky sessions for WebSocket connections.

### FR-DB-007: Monitoring & Metrics
- **Priority:** Must
- The server shall expose Prometheus-compatible metrics.
- Metrics endpoint: `/metrics` on configurable port.
- Key metrics:

| Category | Metrics |
|----------|---------|
| **Server** | uptime, goroutine_count, memory_usage |
| **API** | request_count, request_latency, error_rate (per endpoint) |
| **WebSocket** | active_connections, messages_in, messages_out |
| **Matches** | active_matches, match_create_rate, tick_duration |
| **Matchmaker** | queue_depth, match_formation_rate, avg_wait_time |
| **Database** | query_count, query_latency, connection_pool_usage |
| **Storage** | read_count, write_count, read_latency, write_latency |
| **Auth** | auth_success, auth_failure, token_refresh |
| **Presence** | online_users, presence_events |

### FR-DB-008: Logging
- **Priority:** Must
- Structured logging (JSON format).
- Log levels: DEBUG, INFO, WARN, ERROR.
- Log fields: timestamp, level, message, caller, request_id, user_id.
- Log rotation: configurable (by size, by time).
- Output: stdout (for containerized deployments), file (for traditional).
- Log aggregation compatible (ELK, Loki, CloudWatch).

### FR-DB-009: Admin Console
- **Priority:** Should
- Web-based admin console for server management.
- Features:
  - User management (search, view, ban, unban)
  - Storage browser (view, edit, delete objects)
  - Runtime status and configuration
  - Match list and management
  - Leaderboard management
  - API explorer
  - Server metrics dashboard
- Protected by admin authentication (separate from game auth).
- Default port: 7351.

### FR-DB-010: Health and Readiness
- **Priority:** Must
- Health check endpoints:
  - `/healthcheck` — basic liveness (HTTP 200)
  - Readiness: checks database connectivity, runtime loaded
- Used by:
  - Load balancers
  - Kubernetes probes
  - Monitoring systems

### FR-DB-011: Graceful Shutdown
- **Priority:** Must
- On shutdown signal (SIGTERM, SIGINT):
  1. Stop accepting new connections
  2. Drain active HTTP/gRPC requests
  3. Close WebSocket connections with close frame
  4. Terminate active matches (call `match_terminate`)
  5. Flush pending writes to database
  6. Close database connections
  7. Exit cleanly
- Configurable shutdown timeout (default: 30 seconds).

### FR-DB-012: Data Indexes
- **Priority:** Must
- Optimal indexes for common query patterns:

| Table | Index |
|-------|-------|
| `users` | email, username, custom_id, facebook_id, google_id, etc. |
| `storage` | (collection, key, user_id), (collection, user_id) |
| `leaderboard_record` | (leaderboard_id, expiry_time, score, subscore) |
| `user_edge` | (source_id, state), (destination_id, state) |
| `group_edge` | (source_id, state), (destination_id, state) |
| `notification` | (user_id, create_time) |
| `message` | (channel_id, create_time) |
| `wallet_ledger` | (user_id, create_time) |
---
## 5. Non-Functional Requirements

| Requirement | Specification |
|-------------|---------------|
| **Database** | PostgreSQL 12+ |
| **Availability** | 99.9% uptime (99.99% with multi-node) |
| **Recovery** | RPO < 1 minute, RTO < 5 minutes |
| **Scaling** | Linear horizontal scaling up to 32 nodes |
| **Monitoring** | Prometheus metrics, < 10s scrape interval |
| **Shutdown** | Graceful shutdown ≤ 30 seconds |
| **Deployment** | Docker, Kubernetes, bare metal |
---
## 6. Dependencies

| Dependency | Purpose |
|------------|---------|
| PostgreSQL 12+ | Primary data store |
| Docker | Containerized deployment |
| Kubernetes (optional) | Container orchestration |
| Prometheus | Metrics collection |
| Grafana (optional) | Metrics visualization |
| All feature BRDs | Database schema, queries, data access |
---
## 7. Acceptance Criteria

- [ ] PostgreSQL is the primary data store for all persistent data
- [ ] Schema migrations run automatically on server startup
- [ ] Connection pooling is configurable and monitored
- [ ] Server is configurable via YAML, environment variables, and flags
- [ ] Prometheus metrics are exposed on configurable endpoint
- [ ] Structured logging outputs JSON with proper fields
- [ ] Health check endpoint responds correctly
- [ ] Graceful shutdown completes within timeout
- [ ] Admin console provides user, storage, and match management
- [ ] Multi-node deployment works with shared PostgreSQL
- [ ] Docker image is available and functional
- [ ] Database indexes ensure query performance
---
## 8. Risks & Mitigations

| Risk | Impact | Likelihood | Mitigation |
|------|--------|------------|------------|
| Database single point of failure | Complete outage | Medium | PostgreSQL streaming replication; failover |
| Migration failures | Data corruption | Low | Transactional migrations; backup before migrate |
| Connection pool exhaustion | Request failures | Medium | Monitoring; pool size tuning; PgBouncer |
| Disk space exhaustion | Service disruption | Low | Monitoring alerts; log rotation; data retention |
| Configuration drift (multi-node) | Inconsistent behavior | Medium | Centralized config; configuration management |
| Security vulnerabilities | Data breach | Low | Regular updates; security scanning; TLS everywhere |
---
## 9. Future Considerations

- Read replicas for query scaling
- CockroachDB / YugabyteDB as distributed PostgreSQL alternative
- Redis/Valkey caching layer
- Message queue (NATS, Kafka) for event streaming
- Object storage (S3) for large binary data (replays, assets)
- Edge computing for low-latency regions
- Serverless deployment option
- Infrastructure as Code (Terraform modules)
---
## Version History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2026-07-01 | Engineering | Initial BRD |

---

## Linked Documents
- [PRD-20](../PRD/20_database_infrastructure.md) (API Surface & Interface Specification)
- [TDD-20](../TDD/20_database_infrastructure.md) (Database Schema & Technical Implementation Design)
