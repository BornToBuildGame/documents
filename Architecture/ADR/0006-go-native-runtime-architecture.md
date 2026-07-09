# ADR-0006: Go Native Runtime Architecture

## Status
Accepted

## Date
2026-07-09

## Context

The Ultimate Game Engine server is written in Go, yet the current runtime extensibility layer only supports Lua VM sandboxes (with planned JavaScript/TypeScript VM support). The server's reference architecture supports three runtime languages — **Go, Lua, and TypeScript** — each with distinct execution models.

Go modules are fundamentally different from Lua/JS:
- They are compiled natively as shared object plugins (`.so`) and loaded via `plugin.Open()` at startup.
- They run within the server's own process space with **no VM sandboxing**.
- They receive direct `*sql.DB` database access and full Go ecosystem capabilities.
- They use an `InitModule(ctx, logger, db, nk, initializer)` entry point to register RPCs, hooks, and match handlers.

The current codebase (`internal/runtime/hooks.go`) uses simplified handler signatures that omit `logger`, `db`, and the runtime module interface (`nk`), making it impossible for Go runtime handlers to access the server's subsystems.

## Decision

Implement Go native runtime support using the `.so` plugin loading model with the following architecture:

1. **Plugin Loading**: Go modules are compiled as shared objects using `go build -buildmode=plugin`. The server loads `.so` files from a configured `runtime.path` directory at startup via `plugin.Open()`, resolves the exported `InitModule` symbol, and invokes it.

2. **InitModule Signature**: All Go modules must export:
   ```go
   func InitModule(ctx context.Context, logger runtime.Logger, db *sql.DB, nk runtime.Module, initializer runtime.Initializer) error
   ```

3. **Handler Signatures**: Go RPC, hook, and match handlers receive the full runtime context:
   ```go
   // RPC
   func(ctx context.Context, logger runtime.Logger, db *sql.DB, nk runtime.Module, payload string) (string, error)
   // Before Hook
   func(ctx context.Context, logger runtime.Logger, db *sql.DB, nk runtime.Module, in interface{}) (interface{}, error)
   // After Hook
   func(ctx context.Context, logger runtime.Logger, db *sql.DB, nk runtime.Module, out interface{}, in interface{}) error
   ```

4. **Runtime Precedence**: When the same function name is registered across multiple runtimes, the evaluation order is **Go → Lua → JavaScript**. Go always takes priority.

5. **Safety**: Go modules run without sandbox protection. The server wraps all Go runtime invocations with `recover()` to prevent panics from crashing the server process. For `.so` plugin files, SHA-256 checksum verification against a manifest of approved plugins is recommended.

6. **No Hot Reload**: Go plugins require server restart to reload. This is a known Go plugin limitation.

## Consequences

### Positive
- **Performance**: Go modules run at native speed with zero VM overhead. This is ideal for high-performance match loops, complex matchmaking logic, and latency-critical RPCs.
- **Full Ecosystem Access**: Go modules can import any Go library (HTTP clients, encoding libraries, third-party SDKs).
- **Direct Database Access**: Go handlers receive `*sql.DB`, enabling raw SQL queries and custom transaction patterns beyond the `RuntimeModule` API.
- **Consistency**: The server itself is Go, so Go modules share types, idioms, and tooling.

### Negative
- **No Sandboxing**: Faulty Go code can crash the server process, corrupt memory, or access the filesystem. Panic recovery mitigates crashes but not all failure modes.
- **Go Version Pinning**: `.so` plugins must be compiled with the exact same Go version and compatible dependencies as the server binary. Version mismatches cause load failures.
- **Platform Limitation**: Go's `plugin` package only supports Linux and macOS. Windows is not supported for `.so` plugin loading.
- **No Hot Reload**: Unlike Lua/TypeScript, Go modules cannot be updated without restarting the server.
