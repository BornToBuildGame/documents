# ADR-0001: Use Go as Game Server Framework Language

- **Status**: Accepted
- **Date**: 2026-07-02

---

## Context
The game server framework must handle real-time and session-based multiplayer logic at high scale, supporting low latency (WebSocket packet routing p99 <5ms) and high concurrency (10,000+ connections per node). 
Historically, Node.js/TypeScript and Lua have been popular for game server scripting, but Node.js runs on a single-threaded event loop and Lua VMs can suffer from performance overhead when executing intensive math-heavy game mechanics or managing highly-concurrency goroutine-like routing networks.

## Decision
We will use **Go (Golang)** as the primary programming language for the game server framework core and example handlers.

Go provides:
1. **Lightweight Concurrency**: Goroutines and channels allow simple, O(1) scheduling of thousands of parallel match tick loops.
2. **Native Performance**: Compiles to raw machine code with low memory overhead, avoiding interpreter start-up times.
3. **Robust Ecosystem**: Strong database connection pooling, WebSocket libraries, and gRPC support natively.

TypeScript/JavaScript and Lua will still be supported as **isolated sandbox runtime extensions** for user-defined runtime hooks or authoritative loops, but the core engine and native performance plugins will be authored in Go.

## Consequences
- Developers writing native game server plugins or custom RPC handlers must author them in Go.
- Custom game logic in Lua or TypeScript requires a sandboxed VM bridge (QuickJS/V8 or Lua VM pools), introducing minimal marshalling overhead.
- All code templates and database operation examples in the TDD specifications are standardized to Go.
