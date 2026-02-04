# Durable Objects API

## Class Structure

```typescript
import { DurableObject } from "cloudflare:workers";

export class MyDO extends DurableObject<Env> {
  constructor(ctx: DurableObjectState, env: Env) {
    super(ctx, env);
    // Runs on EVERY wake - keep light!
  }

  // RPC methods (called directly from worker)
  async myMethod(arg: string): Promise<string> { return arg; }

  // fetch handler (legacy/HTTP semantics)
  async fetch(req: Request): Promise<Response> { /* ... */ }

  // Lifecycle handlers
  async alarm() { /* alarm fired */ }
  async webSocketMessage(ws: WebSocket, msg: string | ArrayBuffer) { /* ... */ }
  async webSocketClose(ws: WebSocket, code: number, reason: string, wasClean: boolean) { /* ... */ }
  async webSocketError(ws: WebSocket, error: unknown) { /* ... */ }
}
```

## DurableObjectState Context Methods

### Concurrency Control

```typescript
// Complete work after response sent (e.g., cleanup, logging)
this.ctx.waitUntil(promise: Promise<any>): void

// Critical section - blocks all other requests until complete
await this.ctx.blockConcurrencyWhile(async () => {
  // No other requests processed during this block
  // Use for initialization or critical operations
})
```

**When to use:**
- `waitUntil()`: Background cleanup, logging, non-critical work after response
- `blockConcurrencyWhile()`: First-time init, schema migration, critical state setup

### Lifecycle

```typescript
this.ctx.id              // DurableObjectId of this instance
this.ctx.abort()         // Force eviction (use after PITR restore to reload state)
```

### Storage Access

```typescript
this.ctx.storage.sql     // SQLite API (recommended)
this.ctx.storage.kv      // Sync KV API (SQLite DOs only)
this.ctx.storage         // Async KV API (legacy/KV-only DOs)
```

### WebSocket Management

```typescript
this.ctx.acceptWebSocket(ws: WebSocket, tags?: string[])  // Enable hibernation
this.ctx.getWebSockets(tag?: string): WebSocket[]         // Get by tag or all
this.ctx.getTags(ws: WebSocket): string[]                 // Get tags for connection
```

## Storage APIs

### SQL API (Recommended)

```typescript
const cursor = this.ctx.storage.sql.exec('SELECT * FROM users WHERE email = ?', email);
for (let row of cursor) {} // Objects: { id, name, email }
cursor.toArray(); cursor.one(); // Single row (throws if != 1)
for (let row of cursor.raw()) {} // Arrays: [1, "Alice", "..."]

// Manual iteration
const iter = cursor[Symbol.iterator]();
const first = iter.next(); // { value: {...}, done: false }

cursor.columnNames; // ["id", "name", "email"]
cursor.rowsRead; cursor.rowsWritten; // Billing

type User = { id: number; name: string; email: string };
const user = this.ctx.storage.sql.exec<User>('...', userId).one();
```

### Sync KV API (SQLite DOs only)

```typescript
this.ctx.storage.kv.get("counter"); // undefined if missing
this.ctx.storage.kv.put("counter", 42);
this.ctx.storage.kv.put("user", { name: "Alice", age: 30 });
this.ctx.storage.kv.delete("counter"); // true if existed

for (let [key, value] of this.ctx.storage.kv.list()) {}

// List options: start, prefix, reverse, limit
this.ctx.storage.kv.list({ start: "user:", prefix: "user:", reverse: true, limit: 100 });
```

### Async KV API (Both backends)

```typescript
await this.ctx.storage.get("key"); // Single
await this.ctx.storage.get(["key1", "key2"]); // Multiple (max 128)
await this.ctx.storage.put("key", value); // Single
await this.ctx.storage.put({ "key1": "v1", "key2": { nested: true } }); // Multiple (max 128)
await this.ctx.storage.delete("key");
await this.ctx.storage.delete(["key1", "key2"]);
await this.ctx.storage.list({ prefix: "user:", limit: 100 });

// Options: allowConcurrency, noCache, allowUnconfirmed
await this.ctx.storage.get("key", { allowConcurrency: true, noCache: true });
await this.ctx.storage.put("key", value, { allowUnconfirmed: true, noCache: true });
```

### Storage Options

| Option | Methods | Effect | Use Case |
|--------|---------|--------|----------|
| `allowConcurrency` | get, list | Skip input gate; allow concurrent requests during read | Read-heavy metrics that don't need strict consistency |
| `noCache` | get, put, list | Skip in-memory cache; always read from disk | Rarely-accessed data or testing storage directly |
| `allowUnconfirmed` | put, delete | Return before write confirms (still protected by output gate) | Non-critical writes where latency matters more than confirmation |

## Transactions

```typescript
// Sync (SQL/sync KV only)
this.ctx.storage.transactionSync(() => {
  this.ctx.storage.sql.exec('UPDATE accounts SET balance = balance - ? WHERE id = ?', 100, 1);
  this.ctx.storage.sql.exec('UPDATE accounts SET balance = balance + ? WHERE id = ?', 100, 2);
  return "result";
});

// Async
await this.ctx.storage.transaction(async () => {
  const value = await this.ctx.storage.get("counter");
  await this.ctx.storage.put("counter", value + 1);
  if (value > 100) this.ctx.storage.rollback(); // Explicit rollback
});
```

## Point-in-Time Recovery

```typescript
await this.ctx.storage.getCurrentBookmark();
await this.ctx.storage.getBookmarkForTime(Date.now() - 2 * 24 * 60 * 60 * 1000);
await this.ctx.storage.onNextSessionRestoreBookmark(bookmark);
this.ctx.abort(); // Restart to apply; bookmarks lexically comparable (earlier < later)
```

## Alarms

Schedule future work that survives eviction:

```typescript
// Set alarm (overwrites any existing alarm)
await this.ctx.storage.setAlarm(Date.now() + 3600000)  // 1 hour from now
await this.ctx.storage.setAlarm(new Date("2026-02-01"))  // Absolute time

// Check next alarm
const nextRun = await this.ctx.storage.getAlarm()  // null if none

// Cancel alarm
await this.ctx.storage.deleteAlarm()

// Handler called when alarm fires
async alarm() {
  // Runs once alarm triggers
  // DO wakes from hibernation if needed
  // Use for cleanup, notifications, scheduled tasks
}
```

**Limitations:**
- 1 alarm per DO maximum
- Overwrites previous alarm when set
- Use queue pattern for multiple scheduled events (see [Patterns](./patterns.md))

**Reliability:**
- Alarms survive DO eviction/restart
- Cloudflare retries failed alarms automatically
- Not guaranteed exactly-once (handle idempotently)

## WebSocket Hibernation

Hibernation allows DOs with open WebSocket connections to consume zero compute/memory until message arrives.

```typescript
async fetch(req: Request): Promise<Response> {
  const [client, server] = Object.values(new WebSocketPair());
  this.ctx.acceptWebSocket(server, ["room:123"]);  // Tags for filtering
  server.serializeAttachment({ userId: "abc" });    // Persisted metadata
  return new Response(null, { status: 101, webSocket: client });
}

// Called when message arrives (DO wakes from hibernation)
async webSocketMessage(ws: WebSocket, msg: string | ArrayBuffer) {
  const data = ws.deserializeAttachment();          // Retrieve metadata
  for (const c of this.ctx.getWebSockets("room:123")) c.send(msg);
}

// Called on close (optional handler)
async webSocketClose(ws: WebSocket, code: number, reason: string, wasClean: boolean) {
  // Cleanup logic, remove from lists, etc.
}

// Called on error (optional handler)
async webSocketError(ws: WebSocket, error: unknown) {
  console.error("WebSocket error:", error);
  // Handle error, close connection, etc.
}
```

**Key concepts:**
- **Auto-hibernation:** DO hibernates when no active requests/alarms
- **Zero cost:** Hibernated DOs incur no charges while preserving connections
- **Memory cleared:** All in-memory state lost on hibernation
- **Attachment persistence:** Use `serializeAttachment()` for per-connection metadata that survives hibernation
- **Tags for filtering:** Group connections by room/channel/user for targeted broadcasts

**Handler lifecycle:**
- `webSocketMessage`: DO wakes, processes message, may hibernate after
- `webSocketClose`: Called when client closes (optional - implement for cleanup)
- `webSocketError`: Called on connection error (optional - implement for error handling)

**Metadata persistence:**
```typescript
// Store connection metadata (survives hibernation)
ws.serializeAttachment({ userId: "abc", room: "lobby" })

// Retrieve after hibernation
const { userId, room } = ws.deserializeAttachment()
```

## Misc

```typescript
await this.ctx.storage.deleteAll(); // Atomic for SQLite; alarm NOT included
this.ctx.storage.sql.databaseSize; // Bytes
```

## See Also

- **[Patterns](./patterns.md)** - Real-world usage patterns
- **[Gotchas](./gotchas.md)** - Hibernation caveats, limits, and common errors
- **[Configuration](./configuration.md)** - Wrangler setup and migrations
