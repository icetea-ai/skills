# Durable Objects Patterns

## When to Use Which Pattern

| Need | Pattern | ID Strategy |
|------|---------|-------------|
| Rate limit per user/IP | Rate Limiting | `idFromName(identifier)` |
| Mutual exclusion | Distributed Lock | `idFromName(resource)` |
| >1K req/s throughput | Sharding | `newUniqueId()` or hash |
| Real-time updates | WebSocket Collab | `idFromName(room)` |
| User sessions | Session Management | `idFromName(sessionId)` |
| Background cleanup | Alarm-based | Any |
| Admin + data separation | Control/Data Plane | Control: `idFromName("control")`, Data: `newUniqueId()` |

## RPC vs fetch()

**RPC** (compat >=2024-04-03): Type-safe, simpler, default for new projects
**fetch()**: Legacy compat, HTTP semantics, proxying

```typescript
const count = await stub.increment();  // RPC
const count = await (await stub.fetch(req)).json();  // fetch()
```

## Concurrency Model

### Input/Output Gates

Storage operations automatically block other requests (input gates). Responses wait for writes (output gates).

```typescript
async increment(): Promise<number> {
  // Safe: input gates block interleaving during storage ops
  const val = (await this.ctx.storage.get<number>("count")) ?? 0;
  await this.ctx.storage.put("count", val + 1);
  return val + 1;
}
```

### Write Coalescing

Multiple writes without `await` between them are batched atomically:

```typescript
// Good: All three writes commit atomically
this.ctx.storage.sql.exec("UPDATE accounts SET balance = balance - ? WHERE id = ?", amount, fromId);
this.ctx.storage.sql.exec("UPDATE accounts SET balance = balance + ? WHERE id = ?", amount, toId);
this.ctx.storage.sql.exec("INSERT INTO transfers (from_id, to_id, amount) VALUES (?, ?, ?)", fromId, toId, amount);

// Bad: await breaks coalescing
await this.ctx.storage.put("key1", val1);
await this.ctx.storage.put("key2", val2); // Separate transaction!
```

### Race Conditions with External I/O

`fetch()` and other non-storage I/O allows interleaving:

```typescript
// Race condition possible
async processItem(id: string) {
  const item = await this.ctx.storage.get<Item>(`item:${id}`);
  if (item?.status === "pending") {
    await fetch("https://api.example.com/process"); // Other requests can run here!
    await this.ctx.storage.put(`item:${id}`, { status: "completed" });
  }
}
```

**Solution**: Use optimistic locking (version numbers) or `transaction()`.

### blockConcurrencyWhile()

Blocks ALL concurrency. Use sparingly - only for initialization:

```typescript
// Good: One-time init
constructor(ctx: DurableObjectState, env: Env) {
  super(ctx, env);
  ctx.blockConcurrencyWhile(async () => this.migrate());
}

// Bad: On every request (kills throughput)
async handleRequest() {
  await this.ctx.blockConcurrencyWhile(async () => {
    // ~5ms = max 200 req/sec
  });
}
```

**Never** hold across external I/O (fetch, R2, KV).

## Sharding (High Throughput)

### Throughput by Operation Type

The ~1K req/s limit varies by operation complexity:

| Operation Type | Typical Throughput |
|----------------|-------------------|
| Simple KV read/write | ~1,000 req/s |
| SQLite read | ~500-1,000 req/s |
| SQLite write | ~200-500 req/s |
| Complex queries with JOINs | ~100-300 req/s |
| External I/O (fetch) | Depends on latency |

### Calculating Shard Count

**Formula:** `Required DOs = Total RPS / Capacity per DO`

Example: 10,000 req/s with SQLite writes (~300 req/s per DO)
- Required DOs = 10,000 / 300 = ~34 DOs
- Use 50-100 shards for headroom and even distribution

### Implementation

```typescript
export default {
  async fetch(req: Request, env: Env): Promise<Response> {
    const userId = new URL(req.url).searchParams.get("user");
    const hash = hashCode(userId) % 100;  // 100 shards
    const id = env.COUNTER.idFromName(`shard:${hash}`);
    return env.COUNTER.get(id).fetch(req);
  }
};

function hashCode(str: string): number {
  let hash = 0;
  for (let i = 0; i < str.length; i++) hash = ((hash << 5) - hash) + str.charCodeAt(i);
  return Math.abs(hash);
}
```

**Decisions:**
- **Shard count**: 10-1000 typical (start with 100, measure, adjust)
- **Shard key**: User ID, IP, session - must distribute evenly (use hash)
- **Aggregation**: Coordinator DO or external system (D1, R2)

## Rate Limiting

```typescript
async checkLimit(key: string, limit: number, windowMs: number): Promise<boolean> {
  const req = this.ctx.storage.sql.exec("SELECT COUNT(*) as count FROM requests WHERE key = ? AND timestamp > ?", key, Date.now() - windowMs).one();
  if (req.count >= limit) return false;
  this.ctx.storage.sql.exec("INSERT INTO requests (key, timestamp) VALUES (?, ?)", key, Date.now());
  return true;
}
```

## Distributed Lock

```typescript
private held = false;
async acquire(timeoutMs = 5000): Promise<boolean> {
  if (this.held) return false;
  this.held = true;
  await this.ctx.storage.setAlarm(Date.now() + timeoutMs);
  return true;
}
async release() { this.held = false; await this.ctx.storage.deleteAlarm(); }
async alarm() { this.held = false; }  // Auto-release on timeout
```

## Schema Migrations

### When Do You Need Formal Migration Tracking?

| Operation | Need PRAGMA user_version? |
|-----------|---------------------------|
| CREATE TABLE IF NOT EXISTS | No - inherently idempotent |
| CREATE INDEX IF NOT EXISTS | No - inherently idempotent |
| ADD COLUMN (single) | No - check if column exists first |
| Multiple sequential migrations | Yes - ordering matters |
| Data migration (backfill) | Yes - can't safely re-run |
| DROP COLUMN/TABLE | Yes - destructive, can't undo |
| Rename column/table | Yes - can't check "if renamed" |

### Simple Approach (Single Change)

For adding a single column, check if it exists first. SQLite has no `ADD COLUMN IF NOT EXISTS`:

```typescript
constructor(ctx: DurableObjectState, env: Env) {
  super(ctx, env);
  ctx.blockConcurrencyWhile(async () => {
    this.ctx.storage.sql.exec(`
      CREATE TABLE IF NOT EXISTS items (id INTEGER PRIMARY KEY, data TEXT)
    `);

    // Check before ALTER (SQLite has no ADD COLUMN IF NOT EXISTS)
    const cols = this.ctx.storage.sql.exec("PRAGMA table_info(items)").toArray();
    if (!cols.some(c => c.name === 'status')) {
      this.ctx.storage.sql.exec("ALTER TABLE items ADD COLUMN status TEXT");
    }
  });
}
```

### PRAGMA user_version (Multiple Migrations)

Use `PRAGMA user_version` for incremental schema versioning when you have multiple migrations:

```typescript
constructor(ctx: DurableObjectState, env: Env) {
  super(ctx, env);
  ctx.blockConcurrencyWhile(async () => this.migrate());
}

private async migrate() {
  const version = this.ctx.storage.sql
    .exec<{ user_version: number }>("PRAGMA user_version")
    .one().user_version;

  if (version < 1) {
    this.ctx.storage.sql.exec(`
      CREATE TABLE IF NOT EXISTS items (id INTEGER PRIMARY KEY, data TEXT);
      CREATE INDEX IF NOT EXISTS idx_items_data ON items(data);
      PRAGMA user_version = 1;
    `);
  }
  if (version < 2) {
    this.ctx.storage.sql.exec(`
      ALTER TABLE items ADD COLUMN created_at INTEGER;
      PRAGMA user_version = 2;
    `);
  }
}
```

**Key points:**
- Run migrations in `blockConcurrencyWhile()` during construction
- Each migration block checks `version < N` to run only when needed
- Set `PRAGMA user_version = N` at the end of each migration block
- Migrations are additive - never modify previous migration blocks

### Why Constructor Migrations Work

**DO restarts on deployment:** When you deploy new code, existing DO instances are shut down. The first request after deployment creates a new instance with new code, running the constructor (and migrations).

**Each instance is isolated:** If you have 10,000 DOs, each maintains its own schema version. They migrate independently when accessed.

**Migrations converge lazily:** Unused DOs never migrate (and don't need to). Active DOs migrate on their next cold start.

### Alternative: Per-Request Migrations

Use lazy per-request migrations when:
- Your Worker uses `idFromName()` with potentially untrusted input
- You want `blockConcurrencyWhile()` to not block requests that don't need storage
- You need to validate inside the DO before writing

```typescript
export class ChatRoom extends DurableObject<Env> {
  private migrated = false;

  private ensureMigrated() {
    if (this.migrated) return;

    const version = this.ctx.storage.sql
      .exec<{ user_version: number }>("PRAGMA user_version")
      .one().user_version;

    if (version < 1) {
      this.ctx.storage.sql.exec(`
        CREATE TABLE IF NOT EXISTS messages (id INTEGER PRIMARY KEY, content TEXT);
        PRAGMA user_version = 1;
      `);
    }

    this.migrated = true;
  }

  async sendMessage(userId: string, message: string) {
    this.ensureMigrated();  // Only initializes storage for valid operations
    this.ctx.storage.sql.exec("INSERT INTO messages (content) VALUES (?)", message);
  }

  async getStatus() {
    // No storage needed - no migration triggered, no blockConcurrencyWhile
    return { online: true };
  }
}
```

**Benefits:**
- Storage only initialized for legitimate, validated requests
- Doesn't use `blockConcurrencyWhile()` (which blocks ALL requests, even ones not needing storage)
- Can be called inside transactions naturally

**Trade-offs:**
- Must call `ensureMigrated()` in every method that needs storage
- Slightly more code; easier to forget a call

**When to choose:**

| Situation | Recommendation |
|-----------|----------------|
| Using `idFromString()` or `newUniqueId()` | Constructor migrations (simpler) |
| Using validated `idFromName()` | Constructor migrations (simpler) |
| Using `idFromName()` with untrusted input | Per-request migrations |
| Need non-storage operations to skip blocking | Per-request migrations |

### Deployment Version Mismatch

During deployment rollout, old Worker code may call a DO that already has new code (or vice versa). Handle this in your Worker:

```typescript
async function callDOWithRetry(stub: DurableObjectStub, maxRetries = 3) {
  for (let i = 0; i < maxRetries; i++) {
    try {
      return await stub.myMethod();
    } catch (e: any) {
      if (e.message?.includes('code has been updated')) {
        continue; // Retry - DO will restart with new code
      }
      throw e;
    }
  }
}
```

### Composite Indexes

For queries filtering on multiple columns, create composite indexes:

```typescript
// Query: SELECT * FROM messages WHERE user_id = ? AND created_at > ? ORDER BY created_at
this.ctx.storage.sql.exec(`
  CREATE INDEX IF NOT EXISTS idx_messages_user_time ON messages(user_id, created_at)
`);

// Query: SELECT * FROM orders WHERE status = ? AND priority = ? LIMIT 10
this.ctx.storage.sql.exec(`
  CREATE INDEX IF NOT EXISTS idx_orders_status_priority ON orders(status, priority)
`);
```

**Index design rules:**
- Column order matters: put equality filters first, then range/sort columns
- For `WHERE a = ? AND b > ? ORDER BY b`, index on `(a, b)`
- Covering indexes can include additional columns to avoid table lookups

## In-Memory Caching

```typescript
export class UserCache extends DurableObject<Env> {
  cache = new Map<string, User>();
  async getUser(id: string): Promise<User | undefined> {
    if (this.cache.has(id)) {
      const cached = this.cache.get(id);
      if (cached) return cached;
    }
    const user = await this.ctx.storage.get<User>(`user:${id}`);
    if (user) this.cache.set(id, user);
    return user;
  }
  async updateUser(id: string, data: Partial<User>) {
    const updated = { ...await this.getUser(id), ...data };
    this.cache.set(id, updated);
    await this.ctx.storage.put(`user:${id}`, updated);
    return updated;
  }
}
```

## Batch Processing with Alarms

```typescript
export class BatchProcessor extends DurableObject<Env> {
  pending: string[] = [];
  async addItem(item: string) {
    this.pending.push(item);
    if (!await this.ctx.storage.getAlarm()) await this.ctx.storage.setAlarm(Date.now() + 5000);
  }
  async alarm() {
    const items = [...this.pending];
    this.pending = [];
    this.ctx.storage.sql.exec(
      `INSERT INTO processed_items (item, timestamp) VALUES ${items.map(() => "(?, ?)").join(", ")}`,
      ...items.flatMap(item => [item, Date.now()])
    );
  }
}
```

## Initialization Pattern

```typescript
export class Counter extends DurableObject<Env> {
  value: number;
  constructor(ctx: DurableObjectState, env: Env) {
    super(ctx, env);
    ctx.blockConcurrencyWhile(async () => { this.value = (await ctx.storage.get("value")) || 0; });
  }
  async increment() {
    this.value++;
    this.ctx.storage.put("value", this.value); // Don't await (output gate protects)
    return this.value;
  }
}
```

## Safe Counter / Optimized Write

```typescript
// Input gate blocks other requests
async getUniqueNumber(): Promise<number> {
  let val = await this.ctx.storage.get("counter");
  await this.ctx.storage.put("counter", val + 1);
  return val;
}

// No await on write - output gate delays response until write confirms
async increment(): Promise<Response> {
  let val = await this.ctx.storage.get("counter");
  this.ctx.storage.put("counter", val + 1);
  return new Response(String(val));
}
```

## Parent-Child Coordination

Hierarchical DO pattern where parent manages child DOs:

```typescript
// Parent DO coordinates children
export class Workspace extends DurableObject<Env> {
  async createDocument(name: string): Promise<string> {
    const docId = crypto.randomUUID();
    const childId = this.env.DOCUMENT.idFromName(`${this.ctx.id.toString()}:${docId}`);
    const childStub = this.env.DOCUMENT.get(childId);
    await childStub.initialize(name);

    // Track child in parent storage
    this.ctx.storage.sql.exec('INSERT INTO documents (id, name, created) VALUES (?, ?, ?)',
      docId, name, Date.now());
    return docId;
  }

  async listDocuments(): Promise<string[]> {
    return this.ctx.storage.sql.exec('SELECT id FROM documents').toArray().map(r => r.id);
  }
}

// Child DO
export class Document extends DurableObject<Env> {
  async initialize(name: string) {
    this.ctx.storage.sql.exec('CREATE TABLE IF NOT EXISTS content(key TEXT PRIMARY KEY, value TEXT)');
    this.ctx.storage.sql.exec('INSERT INTO content VALUES (?, ?)', 'name', name);
  }
}
```

## Control Plane / Data Plane Pattern

Separates administrative operations from resource operations for scalability.

### When to Use

- Multi-tenant SaaS (one data plane DO per tenant)
- Document/file management (one data plane DO per document)
- Per-user databases
- Any system where data plane requests >> control plane operations

### Architecture

- **Control Plane**: Single DO manages metadata (create/delete/list resources)
- **Data Plane**: One DO per resource handles actual operations

### Implementation

```typescript
// Control Plane DO - manages all tenants
export class TenantControlPlane extends DurableObject<Env> {
  async createTenant(name: string, locationHint?: DurableObjectLocationHint): Promise<string> {
    const tenantId = this.env.TENANT_DATA.newUniqueId({ locationHint });
    this.ctx.storage.sql.exec(
      "INSERT INTO tenants (id, name, created) VALUES (?, ?, ?)",
      tenantId.toString(), name, Date.now()
    );
    const stub = this.env.TENANT_DATA.get(tenantId);
    await stub.init(name);
    return tenantId.toString();
  }

  async listTenants(): Promise<string[]> {
    return this.ctx.storage.sql.exec("SELECT id FROM tenants").toArray().map(r => r.id);
  }
}

// Data Plane DO - one per tenant
export class TenantData extends DurableObject<Env> {
  async init(name: string) {
    this.ctx.storage.sql.exec("CREATE TABLE IF NOT EXISTS data (key TEXT PRIMARY KEY, value TEXT)");
    this.ctx.storage.sql.exec("INSERT INTO data VALUES ('name', ?)", name);
  }

  async getData(key: string): Promise<string | null> {
    return this.ctx.storage.sql.exec("SELECT value FROM data WHERE key = ?", key).one()?.value ?? null;
  }
}

// Worker routing
export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const url = new URL(request.url);
    if (url.pathname === "/tenants" && request.method === "POST") {
      // Creation goes through control plane
      const control = env.TENANT_CONTROL.getByName("control");
      return Response.json({ id: await control.createTenant("New Tenant") });
    }
    // Data operations go directly to data plane
    const tenantId = url.searchParams.get("tenant");
    const data = env.TENANT_DATA.get(env.TENANT_DATA.idFromString(tenantId!));
    return Response.json(await data.getData("name"));
  }
};
```

### Benefits

- **Performance**: Data requests bypass control plane bottleneck
- **Scalability**: Millions of data plane DOs, each handling its own load
- **Locality**: Use `locationHint` to place data near users

## Hibernation-Aware Pattern

Preserve state across hibernation:

```typescript
async fetch(req: Request): Promise<Response> {
  const [client, server] = Object.values(new WebSocketPair());
  const userId = new URL(req.url).searchParams.get("user");
  server.serializeAttachment({ userId });  // Survives hibernation
  this.ctx.acceptWebSocket(server, ["room:lobby"]);
  server.send(JSON.stringify({ type: "init", state: this.ctx.storage.kv.get("state") }));
  return new Response(null, { status: 101, webSocket: client });
}

async webSocketMessage(ws: WebSocket, msg: string) {
  const { userId } = ws.deserializeAttachment();  // Retrieve after wake
  const state = this.ctx.storage.kv.get("state") || {};
  state[userId] = JSON.parse(msg);
  this.ctx.storage.kv.put("state", state);
  for (const c of this.ctx.getWebSockets("room:lobby")) c.send(msg);
}
```

## Real-time Collaboration

Broadcast updates to all connected clients:

```typescript
async webSocketMessage(ws: WebSocket, msg: string) {
  const data = JSON.parse(msg);
  this.ctx.storage.kv.put("doc", data.content);  // Persist
  for (const c of this.ctx.getWebSockets()) if (c !== ws) c.send(msg);  // Broadcast
}
```

### WebSocket Reconnection

**Client-side** (exponential backoff):
```typescript
class ResilientWS {
  private delay = 1000;
  connect(url: string) {
    const ws = new WebSocket(url);
    ws.onclose = () => setTimeout(() => {
      this.connect(url);
      this.delay = Math.min(this.delay * 2, 30000);
    }, this.delay);
  }
}
```

**Server-side** (cleanup on close):
```typescript
async webSocketClose(ws: WebSocket, code: number, reason: string, wasClean: boolean) {
  const { userId } = ws.deserializeAttachment();
  this.ctx.storage.sql.exec("UPDATE users SET online = false WHERE id = ?", userId);
  for (const c of this.ctx.getWebSockets()) c.send(JSON.stringify({ type: "user_left", userId }));
}
```

## Session Management

```typescript
async createSession(userId: string, data: object): Promise<string> {
  const id = crypto.randomUUID(), exp = Date.now() + 86400000;
  this.ctx.storage.sql.exec("INSERT INTO sessions VALUES (?, ?, ?, ?)", id, userId, JSON.stringify(data), exp);
  await this.ctx.storage.setAlarm(exp);
  return id;
}

async getSession(id: string): Promise<object | null> {
  const row = this.ctx.storage.sql.exec("SELECT data FROM sessions WHERE id = ? AND expires_at > ?", id, Date.now()).one();
  return row ? JSON.parse(row.data) : null;
}

async alarm() { this.ctx.storage.sql.exec("DELETE FROM sessions WHERE expires_at <= ?", Date.now()); }
```

## Multiple Events (Single Alarm)

Queue pattern to schedule multiple events:

```typescript
async scheduleEvent(id: string, runAt: number) {
  await this.ctx.storage.put(`event:${id}`, { id, runAt });
  const curr = await this.ctx.storage.getAlarm();
  if (!curr || runAt < curr) await this.ctx.storage.setAlarm(runAt);
}

async alarm() {
  const events = await this.ctx.storage.list({ prefix: "event:" }), now = Date.now();
  let next = null;
  for (const [key, ev] of events) {
    if (ev.runAt <= now) {
      await this.processEvent(ev);
      await this.ctx.storage.delete(key);
    } else if (!next || ev.runAt < next) next = ev.runAt;
  }
  if (next) await this.ctx.storage.setAlarm(next);
}
```

## Alarm Idempotency (CRITICAL)

**Alarms may fire more than once.** The runtime guarantees at-least-once delivery, not exactly-once. Always design alarm handlers to be idempotent.

```typescript
// WRONG - not idempotent, double-charges on retry
async alarm() {
  const subscription = await this.ctx.storage.get<Subscription>("sub");
  await this.chargeUser(subscription.userId, subscription.amount);
  await this.ctx.storage.setAlarm(Date.now() + 30 * 24 * 60 * 60 * 1000);
}

// RIGHT - idempotent with guard
async alarm() {
  const subscription = await this.ctx.storage.get<Subscription>("sub");

  // Check if already processed this billing cycle
  if (subscription.lastBilledAt >= this.getBillingCycleStart()) {
    // Already billed, just reschedule
    await this.ctx.storage.setAlarm(this.getNextBillingDate());
    return;
  }

  await this.chargeUser(subscription.userId, subscription.amount);

  // Mark as processed BEFORE rescheduling
  await this.ctx.storage.put("sub", {
    ...subscription,
    lastBilledAt: Date.now()
  });

  await this.ctx.storage.setAlarm(this.getNextBillingDate());
}
```

**Idempotency patterns:**
- Store "processed" flag or timestamp before taking action
- Use unique transaction IDs for external calls
- Check current state before modifying

## Graceful Cleanup

Use `ctx.waitUntil()` to complete work after response:

```typescript
async myMethod() {
  const response = { success: true };
  this.ctx.waitUntil(this.ctx.storage.sql.exec("DELETE FROM old_data WHERE timestamp < ?", cutoff));
  return response;
}
```

## Cleanup

```typescript
async cleanup() {
  await this.ctx.storage.deleteAlarm(); // Separate from deleteAll
  await this.ctx.storage.deleteAll();
}
```

## Best Practices

- **Design**: Use `idFromName()` for coordination, `newUniqueId()` for sharding, minimize constructor work
- **Storage**: Prefer SQLite, batch with transactions, set alarms for cleanup, use PITR before risky ops
- **Performance**: ~1K req/s per DO max - shard for more, cache in memory, use alarms for deferred work
- **Reliability**: Handle 503 with retry+backoff, design for cold starts, test migrations with `--dry-run`
- **Security**: Validate inputs in Workers, **prefer `idFromString()` or factory patterns over `idFromName()` with untrusted input** (prevents leaked DOs), rate limit DO creation, use jurisdiction for compliance

## See Also

- **[API](./api.md)** - ctx methods, WebSocket handlers
- **[Gotchas](./gotchas.md)** - Hibernation caveats, common errors
- **[Testing](./testing.md)** - Testing patterns with Vitest
