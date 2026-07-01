# BRD-12: Storage Engine

> **Project:** Ultimate Game Engine — Multiplayer Game Server  
> **Feature:** Storage Engine  
> **Version:** 1.0  
> **Last Updated:** 2026-07-01  
> **Status:** Draft  
> **Priority:** Must Have
---
## 1. Purpose

Define the requirements for a NoSQL-style document storage layer that enables developers to store and retrieve arbitrary JSON data per user or globally. This system stores player inventory, settings, quest progress, character data, cosmetics, and any game-specific data.
---
## 2. Scope

### In Scope
- Document storage with collection/key/user namespace
- JSON document read/write
- Permission model (public, private, server-only)
- Conditional writes (optimistic concurrency via version checking)
- Batch reads and writes
- Collection listing
- Global (server-owned) and per-user storage
- Storage Index Search (querying and searching JSON fields)

### Out of Scope
- Direct relational database queries from clients (use custom RPCs)
- Binary object/file hosting (such as game builds, textures)
- Storage quotas and direct billing
---
## 3. Stakeholders

| Role | Interest |
|------|----------|
| Game Developers | Flexible data storage without schema changes |
| Players | Persistent game progress and inventory |
| Game Designers | Store game content and configuration |
| Server Administrators | Data management, backup, migration |
---
## 4. Functional Requirements

### FR-ST-001: Document Model
- **Priority:** Must
- Each document is identified by a composite key:
  - `collection`: string (namespace, e.g., `"inventory"`, `"settings"`)
  - `key`: string (document identifier, e.g., `"player_loadout"`)
  - `user_id`: UUID (owner; empty for global/server-owned documents)
- Document value: JSON object (max size: configurable, default: 256 KB).
- Each document has a `version` string for concurrency control.

### FR-ST-002: Write Operations
- **Priority:** Must
- Write a single document or batch of documents.
- Write request fields:
  - `collection`, `key`, `user_id` (optional), `value` (JSON), `version` (optional)
  - `permission_read`: int (0=no read, 1=owner only, 2=public)
  - `permission_write`: int (0=no write, 1=owner only)
- If `version` is provided:
  - Conditional write: only succeeds if current version matches (optimistic concurrency).
  - Returns error if version mismatch.
- If `version` is empty:
  - Unconditional write (overwrite or create).
- Successful write returns the new version.

### FR-ST-003: Read Operations
- **Priority:** Must
- Read a single document or batch of documents.
- Read request: `collection`, `key`, `user_id`.
- Returns: `collection`, `key`, `user_id`, `value`, `version`, `permission_read`, `permission_write`, `create_time`, `update_time`.
- Permission enforcement:
  - `permission_read = 1`: only the owner can read
  - `permission_read = 2`: any authenticated user can read
  - Server runtime can read regardless of permissions.

### FR-ST-004: Delete Operations
- **Priority:** Must
- Delete a single document or batch.
- Conditional delete with version check supported.
- Only the owner or server runtime can delete.

### FR-ST-005: List Operations
- **Priority:** Must
- List documents within a collection (optionally filtered by user_id).
- Parameters:
  - `collection`: required
  - `user_id`: optional (filter by owner)
  - `limit`: max results (default: 10, max: 100)
  - `cursor`: pagination token
- Results include full document data.

### FR-ST-006: Permission Model
- **Priority:** Must
- Two permission dimensions:
  - **Read permission**: `0` (no read), `1` (owner only), `2` (public/any user)
  - **Write permission**: `0` (no write / server only), `1` (owner only)
- Server runtime always has full access regardless of permissions.
- Default permissions: read=1 (owner), write=1 (owner).

### FR-ST-007: Version Checking (Optimistic Concurrency)
- **Priority:** Must
- Each document has a version string (hash/UUID) that changes on every write.
- Conditional writes require matching the current version.
- Prevents race conditions when multiple clients update the same document.
- Version mismatch returns a conflict error (HTTP 409 / gRPC ABORTED).

### FR-ST-008: Global/Server Storage
- **Priority:** Should
- Documents with no user_id are "global" (server-owned).
- Only server runtime can write global documents.
- Global documents can be publicly readable.
- Use cases: game configuration, event schedules, server announcements.

### FR-ST-009: Storage Hooks
- **Priority:** Should
- Server runtime hooks:
- `before_storage_write`: validate/modify data before writing
- `after_storage_write`: trigger side effects
- `before_storage_read`: filter/restrict access
- `before_storage_delete`: validate deletion

### FR-ST-010: Storage Index Search
- **Priority:** Should
- The storage system shall support indexing and querying of JSON document fields.
- Query syntax: Lucene/Bleve-like query syntax (e.g., `+value.level:>=5 +value.class:warrior`).
- Developers can query storage objects within a specific collection, optionally filtered by user ID.
- Search queries shall be paginated via limit and cursor tokens.
---
## 5. Non-Functional Requirements

| Requirement | Specification |
|-------------|---------------|
| **Performance** | Read ≤ 50ms (p99), Write ≤ 100ms (p99) |
| **Scalability** | Support billions of documents |
| **Document Size** | Max 256 KB per document (configurable) |
| **Batch Size** | Max 128 operations per batch |
| **Availability** | 99.9% uptime |
| **Durability** | Documents persisted to PostgreSQL with WAL |
| **Consistency** | Strong consistency for reads after writes |
---
## 6. Dependencies

| Dependency | Purpose |
|------------|---------|
| BRD-01: User Authentication | Document ownership identity |
| BRD-18: Server Runtime & Hooks | Storage event hooks |
| BRD-04: Authoritative Game Server | Server-side storage access |
| PostgreSQL | Persistent document storage |
---
## 7. Acceptance Criteria

- [ ] Documents can be written with collection/key/user_id composite key
- [ ] Documents can be read individually and in batches
- [ ] Permission model enforces read/write access correctly
- [ ] Version-based conditional writes prevent race conditions
- [ ] Version mismatch returns appropriate error
- [ ] Collection listing supports pagination
- [ ] Global (server-owned) documents work without user_id
- [ ] Server runtime has unrestricted access to all documents
- [ ] Storage hooks fire on read/write/delete operations
- [ ] Performance: read ≤ 50ms, write ≤ 100ms (p99)
---
## 8. Risks & Mitigations

| Risk | Impact | Likelihood | Mitigation |
|------|--------|------------|------------|
| Large document abuse | Storage/memory pressure | Medium | Size limits; per-user document count limits |
| Hot key contention | Write conflicts | Medium | Optimistic concurrency; retry guidance |
| Schema evolution (JSON) | Data compatibility issues | Medium | Version fields in data; migration hooks |
| Unbounded collection growth | Query performance | Low | Pagination limits; TTL for ephemeral data |
| Client-side data tampering | Game state corruption | High | Use permission_write=0; server-only writes |
---
## 9. Future Considerations

- Secondary indexes on JSON fields
- Full-text search within documents
- Document TTL (auto-expiry)
- Change data capture (stream changes to external systems)
- Document schema validation
- Encryption at rest for sensitive data
- Cross-collection transactions
---
## Version History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2026-07-01 | Engineering | Initial BRD |

---

## Linked Documents
- [PRD-12](../PRD/12_storage_engine.md) (API Surface & Interface Specification)
- [TDD-12](../TDD/12_storage_engine.md) (Database Schema & Technical Implementation Design)
