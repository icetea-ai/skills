---
name: durable-objects
description: "Use when working with Cloudflare Durable Objects - reviewing, creating, or modifying DO code. Triggers on: extends DurableObject, DurableObjectState, ctx.storage, blockConcurrencyWhile, alarms, WebSockets, RPC methods, SQLite storage, stateful coordination."
references:
  - api
  - configuration
  - patterns
  - testing
  - gotchas
  - workers-integration
---

# Durable Objects

Build stateful, coordinated applications on Cloudflare's edge using Durable Objects.

## Reading Order

1. **[Configuration](./references/configuration.md)** - wrangler.jsonc setup, migrations, bindings
2. **[API](./references/api.md)** - Class structure, storage APIs (SQL, KV), alarms, WebSockets
3. **[Patterns](./references/patterns.md)** - Concurrency, sharding, rate limiting, real-time collaboration
4. **[Gotchas](./references/gotchas.md)** - **Read the concurrency gates section first!** Limits, common errors
5. **[Testing](./references/testing.md)** - Vitest setup, `runInDurableObject()`, alarm testing
6. **[Workers Integration](./references/workers-integration.md)** - Handler patterns, validation, logging

## When to Use

- Creating new Durable Object classes for stateful coordination
- Implementing RPC methods, alarms, or WebSocket handlers
- Reviewing existing DO code for best practices
- Configuring wrangler.jsonc/toml for DO bindings and migrations

## Use Durable Objects For

| Need | Example |
|------|---------|
| Coordination | Chat rooms, multiplayer games, collaborative docs |
| Strong consistency | Inventory, booking systems, turn-based games |
| Per-entity storage | Multi-tenant SaaS, per-user data |
| Persistent connections | WebSockets, real-time notifications |
| Scheduled work per entity | Subscription renewals, game timeouts |

**Quick test:** If your database queries consistently filter by `WHERE resourceID = 'abc'`, DOs likely simplify your architecture.

## Do NOT Use For

- Stateless request handling (use plain Workers)
- Maximum global distribution needs
- High fan-out independent requests

## State Management Layers

| Layer | Type | Durability | Speed | Use Case |
|-------|------|------------|-------|----------|
| In-memory variables | Transient | Lost on eviction | Fastest | Caches, computed values |
| `ctx.storage.sql` | Persistent | Survives eviction | Fast | Primary data, relationships |
| `ctx.storage` KV | Persistent | Survives eviction | Fast | Simple key-value data |
| `ws.serializeAttachment()` | Per-connection | Survives hibernation | Fast | WebSocket session metadata |
| External (R2, D1, KV) | Global | Globally durable | Slower | Shared state, large objects |

**Rule:** Always persist critical state to storage. Use in-memory only for caching data that can be reconstructed.

## Quick Reference

### Wrangler Configuration

```jsonc
// wrangler.jsonc
{
  "durable_objects": {
    "bindings": [{ "name": "MY_DO", "class_name": "MyDurableObject" }]
  },
  "migrations": [{ "tag": "v1", "new_sqlite_classes": ["MyDurableObject"] }]
}
```

### Basic Durable Object Pattern

```typescript
import { DurableObject } from "cloudflare:workers";

export interface Env {
  MY_DO: DurableObjectNamespace<MyDurableObject>;
}

export class MyDurableObject extends DurableObject<Env> {
  constructor(ctx: DurableObjectState, env: Env) {
    super(ctx, env);
    ctx.blockConcurrencyWhile(async () => {
      this.ctx.storage.sql.exec(`
        CREATE TABLE IF NOT EXISTS items (
          id INTEGER PRIMARY KEY AUTOINCREMENT,
          data TEXT NOT NULL
        )
      `);
    });
  }

  async addItem(data: string): Promise<number> {
    const result = this.ctx.storage.sql.exec<{ id: number }>(
      "INSERT INTO items (data) VALUES (?) RETURNING id",
      data
    );
    return result.one().id;
  }
}

export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const stub = env.MY_DO.getByName("my-instance");
    const id = await stub.addItem("hello");
    return Response.json({ id });
  },
};
```

## Critical Rules

1. **Model around coordination atoms** - One DO per chat room/game/user, not one global DO
2. **Use `getByName()` for deterministic routing** - Same input = same DO instance
3. **Use SQLite storage** - Configure `new_sqlite_classes` in migrations
4. **Initialize in constructor** - Use `blockConcurrencyWhile()` for schema setup only
5. **Use RPC methods** - Not fetch() handler (compatibility date >= 2024-04-03)
6. **Persist first, cache second** - Always write to storage before updating in-memory state
7. **One alarm per DO** - `setAlarm()` replaces any existing alarm

## Anti-Patterns (NEVER)

- Single global DO handling all requests (bottleneck)
- Using `blockConcurrencyWhile()` on every request (kills throughput)
- Storing critical state only in memory (lost on eviction/crash)
- Using `await` between related storage writes (breaks atomicity)
- Holding `blockConcurrencyWhile()` across `fetch()` or external I/O

## Stub Creation

```typescript
// Deterministic - preferred for most cases
const stub = env.MY_DO.getByName("room-123");

// From existing ID string
const id = env.MY_DO.idFromString(storedIdString);
const stub = env.MY_DO.get(id);

// New unique ID - store mapping externally
const id = env.MY_DO.newUniqueId();
const stub = env.MY_DO.get(id);
```

## Storage Operations

```typescript
// SQL (synchronous, recommended)
this.ctx.storage.sql.exec("INSERT INTO t (c) VALUES (?)", value);
const rows = this.ctx.storage.sql.exec<Row>("SELECT * FROM t").toArray();

// KV (async)
await this.ctx.storage.put("key", value);
const val = await this.ctx.storage.get<Type>("key");
```

## Alarms

```typescript
// Schedule (replaces existing)
await this.ctx.storage.setAlarm(Date.now() + 60_000);

// Handler
async alarm(): Promise<void> {
  // Process scheduled work
  // Optionally reschedule: await this.ctx.storage.setAlarm(...)
}

// Cancel
await this.ctx.storage.deleteAlarm();
```

## Schema Migrations

```typescript
// Simple: Single column addition (check if exists)
const cols = this.ctx.storage.sql.exec("PRAGMA table_info(items)").toArray();
if (!cols.some(c => c.name === 'status')) {
  this.ctx.storage.sql.exec("ALTER TABLE items ADD COLUMN status TEXT");
}

// Complex: Multiple migrations (use KV-based version tracking)
const version = this.ctx.storage.kv.get("__schema_version") ?? 0;
if (version < 1) { /* migration 1 */ }
if (version < 2) { /* migration 2 */ this.ctx.storage.kv.put("__schema_version", 2); }
```

> **Warning:** `PRAGMA user_version` returns `not authorized` in `wrangler dev` local mode. Use KV-based tracking instead.

See [Patterns: Schema Migrations](./references/patterns.md#schema-migrations) for decision tree and details.

## Testing Quick Start

```typescript
import { env } from "cloudflare:test";
import { describe, it, expect } from "vitest";

describe("MyDO", () => {
  it("should work", async () => {
    const stub = env.MY_DO.getByName("test");
    const result = await stub.addItem("test");
    expect(result).toBe(1);
  });
});
```
