# Durable Objects Gotchas

## Concurrency Model (CRITICAL - Read First!)

Durable Objects use **input/output gates** to prevent race conditions. Understanding these gates is essential for correct DO code.

### Input Gates

Block new requests during storage reads from CURRENT request:

```typescript
// SAFE: Input gate active during await
async increment() {
  const val = await this.ctx.storage.get("counter"); // Input gate blocks other requests
  await this.ctx.storage.put("counter", val + 1);
  return val;
}
```

### Output Gates

Hold response until ALL writes from current request confirm:

```typescript
// SAFE: Output gate waits for put() to confirm before returning response
async increment() {
  const val = await this.ctx.storage.get("counter");
  this.ctx.storage.put("counter", val + 1); // No await
  return new Response(String(val)); // Response delayed until write confirms
}
```

### Write Coalescing

Multiple writes to same key = atomic (last write wins):

```typescript
// SAFE: All three writes coalesce atomically
this.ctx.storage.put("key", 1);
this.ctx.storage.put("key", 2);
this.ctx.storage.put("key", 3); // Final value: 3
```

### Breaking Gates (DANGER!)

**`fetch()` breaks input/output gates** - allows request interleaving:

```typescript
// UNSAFE: fetch() allows another request to interleave
async unsafe() {
  const val = await this.ctx.storage.get("counter");
  await fetch("https://api.example.com"); // Gate broken!
  await this.ctx.storage.put("counter", val + 1); // Race condition possible
}
```

**Solution:** Use `blockConcurrencyWhile()` or `transaction()`:

```typescript
// SAFE: Block concurrent requests explicitly
async safe() {
  return await this.ctx.blockConcurrencyWhile(async () => {
    const val = await this.ctx.storage.get("counter");
    await fetch("https://api.example.com");
    await this.ctx.storage.put("counter", val + 1);
    return val;
  });
}
```

### allowConcurrency Option

Opt out of input gate for reads that don't need protection:

```typescript
// Allow concurrent reads (no consistency guarantee)
const val = await this.ctx.storage.get("metrics", { allowConcurrency: true });
```

## Common Errors

### "Hibernation Cleared My In-Memory State"

**Problem:** Variables lost after hibernation
**Cause:** DO auto-hibernates when idle; in-memory state not persisted
**Solution:** Use `ctx.storage` for critical data, `ws.serializeAttachment()` for per-connection metadata

```typescript
// Wrong - lost on hibernation
private userCount = 0;
async webSocketMessage(ws: WebSocket, msg: string) {
  this.userCount++;  // Lost!
}

// Right - persisted
async webSocketMessage(ws: WebSocket, msg: string) {
  const count = this.ctx.storage.kv.get("userCount") || 0;
  this.ctx.storage.kv.put("userCount", count + 1);
}
```

### "setTimeout Didn't Fire After Restart"

**Problem:** Scheduled work lost on eviction
**Cause:** `setTimeout` in-memory only; eviction clears timers
**Solution:** Use `ctx.storage.setAlarm()` for reliable scheduling

```typescript
// Wrong - lost on eviction
setTimeout(() => this.cleanup(), 3600000);

// Right - survives eviction
await this.ctx.storage.setAlarm(Date.now() + 3600000);
async alarm() { await this.cleanup(); }
```

### "Constructor Runs on Every Wake"

**Problem:** Expensive init logic slows all requests
**Cause:** Constructor runs on every wake (first request after eviction OR after hibernation)
**Solution:** Lazy initialization or cache in storage

**Critical understanding:** Constructor runs in two scenarios:
1. **Cold start** - DO evicted from memory, first request creates new instance
2. **Wake from hibernation** - DO with WebSockets hibernated, message/alarm wakes it

```typescript
// Wrong - expensive on every wake
constructor(ctx: DurableObjectState, env: Env) {
  super(ctx, env);
  this.heavyData = this.loadExpensiveData();  // Slow!
}

// Right - lazy load
private heavyData?: HeavyData;
private getHeavyData() {
  if (!this.heavyData) this.heavyData = this.loadExpensiveData();
  return this.heavyData;
}
```

### "Migration Runs on Every Wake (That's OK)"

**Problem:** Concerned that migrations run repeatedly
**Cause:** Constructor runs on every cold start, so migration code executes each time
**Reality:** This is expected and correct behavior

**Key insight:** Migrations MUST be idempotent. They check state before acting:

```typescript
// OK: CREATE TABLE IF NOT EXISTS is idempotent
this.ctx.storage.sql.exec("CREATE TABLE IF NOT EXISTS items (...)");

// OK: PRAGMA user_version check skips completed migrations
const v = this.ctx.storage.sql.exec("PRAGMA user_version").one().user_version;
if (v < 2) { /* only runs if needed */ }

// WRONG: ALTER TABLE ADD COLUMN fails if column exists
this.ctx.storage.sql.exec("ALTER TABLE items ADD COLUMN status TEXT"); // Error on re-run!

// RIGHT: Check column existence first
const cols = this.ctx.storage.sql.exec("PRAGMA table_info(items)").toArray();
if (!cols.some(c => c.name === 'status')) {
  this.ctx.storage.sql.exec("ALTER TABLE items ADD COLUMN status TEXT");
}
```

**Why this works with deployments:** When you deploy new code, existing DO instances are shut down. The first request after deployment creates a new instance with new code. So your migration runs with the new code, ensuring schema and code stay in sync.

### "Silent Data Corruption with Large IDs"

**Cause:** JavaScript numbers have 53-bit precision; SQLite INTEGER is 64-bit
**Symptom:** IDs > 9007199254740991 (Number.MAX_SAFE_INTEGER) silently truncate/corrupt
**Solution:** Store large IDs as TEXT:

```typescript
// BAD: Snowflake/Twitter IDs will corrupt
this.ctx.storage.sql.exec("CREATE TABLE events(id INTEGER PRIMARY KEY)");
this.ctx.storage.sql.exec("INSERT INTO events VALUES (?)", 1234567890123456789n); // Corrupts!

// GOOD: Store as TEXT
this.ctx.storage.sql.exec("CREATE TABLE events(id TEXT PRIMARY KEY)");
this.ctx.storage.sql.exec("INSERT INTO events VALUES (?)", "1234567890123456789");
```

### "Durable Object Overloaded (503 errors)"

**Problem:** 503 errors under load
**Cause:** Single DO exceeding ~1K req/s throughput limit
**Solution:** Shard across multiple DOs (see [Patterns: Sharding](./patterns.md))

### "Storage Quota Exceeded (Write failures)"

**Problem:** Write operations failing
**Cause:** DO storage exceeding 10GB limit or account quota
**Solution:** Cleanup with alarms, use `deleteAll()` for old data, upgrade plan

### "CPU Time Exceeded (Terminated)"

**Problem:** Request terminated mid-execution
**Cause:** Processing exceeding 30s CPU time default limit
**Solution:** Increase `limits.cpu_ms` in wrangler.jsonc (max 300s) or chunk work

### "WebSockets Disconnect on Eviction"

**Problem:** Connections drop unexpectedly
**Cause:** DO evicted from memory without hibernation API
**Solution:** Use WebSocket hibernation handlers + client reconnection logic

### "Migration Failed (Deploy error)"

**Cause:** Non-unique tags, non-sequential tags, or invalid class names in migration
**Solution:** Check tag uniqueness/sequential ordering and verify class names are correct

### "RPC Method Not Found"

**Cause:** compatibility_date < 2024-04-03 preventing RPC usage
**Solution:** Update compatibility_date to >= 2024-04-03 or use fetch() instead of RPC

### "Only One Alarm Allowed"

**Cause:** Need multiple scheduled tasks but only one alarm supported per DO
**Solution:** Use event queue pattern to schedule multiple tasks with single alarm

### "Race Condition Despite Single-Threading"

**Problem:** Concurrent requests see inconsistent state
**Cause:** Async operations allow request interleaving (await = yield point)
**Solution:** Use `blockConcurrencyWhile()` for critical sections or atomic storage ops

```typescript
// Wrong - race condition
async incrementCounter() {
  const count = await this.ctx.storage.get("count") || 0;
  // Another request could execute here during await
  await this.ctx.storage.put("count", count + 1);
}

// Right - atomic operation
async incrementCounter() {
  return this.ctx.storage.sql.exec(
    "INSERT INTO counters (id, value) VALUES (1, 1) ON CONFLICT(id) DO UPDATE SET value = value + 1 RETURNING value"
  ).one().value;
}

// Right - explicit locking
async criticalOperation() {
  await this.ctx.blockConcurrencyWhile(async () => {
    const count = await this.ctx.storage.get("count") || 0;
    await this.ctx.storage.put("count", count + 1);
  });
}
```

### "Race Condition in Concurrent Calls"

**Cause:** Multiple concurrent storage operations initiated from same event (e.g., `Promise.all()`) are not protected by input gate
**Solution:** Avoid concurrent storage operations within single event; input gate only serializes requests from different events, not operations within same event

### "DO Doesn't Know Its Own ID"

**Problem:** Need to access the DO's name/ID from inside the DO
**Cause:** DOs don't have access to their own name or ID internally - only the caller knows it
**Solution:** Explicitly pass identity via an `init()` method and store in storage

```typescript
export class MyDurableObject extends DurableObject<Env> {
  private name?: string;

  async init(name: string) {
    // Store name if not already set
    const existing = await this.ctx.storage.get<string>("_name");
    if (!existing) {
      await this.ctx.storage.put("_name", name);
    }
    this.name = existing ?? name;
  }

  // Call from Worker after getting stub
  // const stub = env.MY_DO.getByName("user-123");
  // await stub.init("user-123");
}
```

### "Unawaited RPC Call Failed Silently"

**Problem:** RPC errors swallowed, return values lost
**Cause:** Not awaiting RPC stub method calls
**Solution:** Always `await` RPC calls - they return Promises even if the method signature suggests otherwise

```typescript
// WRONG - error swallowed, return value lost
stub.myMethod(data);

// RIGHT - errors propagate, value captured
const result = await stub.myMethod(data);
```

### "Leaked DOs from Unvalidated idFromName()"

**Problem:** DOs with initialized storage created by unauthorized/brute-forced requests
**Cause:** `idFromName(untrustedInput)` accepts any string, creating DOs for invalid identifiers
**Solution:** Validate before routing, use `idFromString()`, or use per-request migrations

The risk depends on your ID strategy:

| Strategy | Risk Level | Notes |
|----------|------------|-------|
| `idFromName(untrustedInput)` | **High** | Any string creates a valid DO ID |
| `idFromString(untrustedInput)` | **Low** | Fails if not a valid, existing DO ID |
| `newUniqueId()` | **None** | You control creation entirely |
| Factory/lookup pattern | **None** | User provides code → you lookup real DO ID |

**Risky pattern:**

```typescript
// RISKY: idFromName accepts any string - attacker can create arbitrary DOs
const roomId = url.searchParams.get("room");  // Attacker-controlled: "hacked123"
const id = env.CHAT_ROOM.idFromName(roomId);  // Valid ID created for ANY string
const stub = env.CHAT_ROOM.get(id);           // DO created, constructor runs
await stub.sendMessage(body);                 // Validation happens too late
```

**Safe patterns:**

```typescript
// SAFE: idFromString fails for invalid IDs
const doIdString = url.searchParams.get("id");      // Must be valid 64-char hex
const id = env.CHAT_ROOM.idFromString(doIdString);  // Throws if invalid!
const stub = env.CHAT_ROOM.get(id);

// SAFE: Factory/lookup pattern (voucher example)
const voucherCode = url.searchParams.get("code");   // Human-readable: "SAVE20"
const voucher = await env.DB.prepare(
  "SELECT do_id FROM vouchers WHERE code = ?"
).bind(voucherCode).first();
if (!voucher) return Response.json({ error: "Invalid voucher" }, { status: 404 });
const id = env.VOUCHER.idFromString(voucher.do_id); // Known-valid ID
const stub = env.VOUCHER.get(id);

// SAFE: Validate before idFromName
const roomId = url.searchParams.get("room");
const isValidRoom = await env.DB.prepare(
  "SELECT 1 FROM rooms WHERE id = ?"
).bind(roomId).first();
if (!isValidRoom) return Response.json({ error: "Room not found" }, { status: 404 });
const stub = env.CHAT_ROOM.getByName(roomId);  // Now safe - room exists
```

For cases where you must use `idFromName()` with potentially untrusted input, consider [per-request migrations](./patterns.md#alternative-per-request-migrations) so storage isn't initialized until after validation inside the DO.

### "WebSocket 1006 Error on Close"

**Problem:** Client gets 1006 (abnormal closure) instead of clean close code
**Cause:** `webSocketClose` handler didn't reciprocate the close
**Solution:** Call `ws.close()` in the handler to complete the closing handshake

```typescript
async webSocketClose(ws: WebSocket, code: number, reason: string, wasClean: boolean) {
  // CRITICAL: Reciprocate the close to avoid 1006 error
  ws.close(code, reason);

  // Then do cleanup
  const { userId } = ws.deserializeAttachment();
  this.ctx.storage.sql.exec("UPDATE users SET online = false WHERE id = ?", userId);
}
```

### "Can't Retry DO Errors Properly"

**Problem:** Need to implement retry logic for transient failures
**Cause:** Not checking error properties for retry hints
**Solution:** Check `error.retryable` and `error.overloaded` properties, use exponential backoff

```typescript
async callDOWithRetry(stub: DurableObjectStub, maxRetries = 3) {
  for (let attempt = 0; attempt < maxRetries; attempt++) {
    try {
      return await stub.myMethod();
    } catch (error: any) {
      if (error.retryable || error.overloaded) {
        // Exponential backoff: 100ms, 200ms, 400ms...
        const delay = Math.min(100 * Math.pow(2, attempt), 5000);
        await new Promise(r => setTimeout(r, delay));
        continue;
      }
      throw error; // Non-retryable error
    }
  }
  throw new Error("Max retries exceeded");
}
```

### "Lost State on Unexpected Shutdown"

**Problem:** Data lost when DO shuts down unexpectedly
**Cause:** Relying on shutdown hooks (which don't exist) or buffering state in memory
**Solution:** Write state incrementally as you process; design for crash-safety. There are no shutdown hooks in DOs.

```typescript
// WRONG - buffering state, lost on crash
async processItems(items: Item[]) {
  const results = [];
  for (const item of items) {
    results.push(await this.process(item));
  }
  // If crash happens here, all work is lost!
  await this.ctx.storage.put("results", results);
}

// RIGHT - write incrementally
async processItems(items: Item[]) {
  for (const item of items) {
    const result = await this.process(item);
    // Persisted immediately - survives crash
    this.ctx.storage.sql.exec(
      "INSERT INTO results (item_id, data) VALUES (?, ?)",
      item.id, JSON.stringify(result)
    );
  }
}
```

### "Direct SQL Transaction Statements"

**Cause:** Using `BEGIN TRANSACTION` directly instead of transaction methods
**Solution:** Use `this.ctx.storage.transactionSync()` for sync operations or `this.ctx.storage.transaction()` for async operations

### "Async in transactionSync"

**Cause:** Using async operations inside `transactionSync()` callback
**Solution:** Use async `transaction()` method instead of `transactionSync()` when async operations needed

### "Alarm Not Deleted with deleteAll()"

**Cause:** `deleteAll()` doesn't delete alarms automatically
**Solution:** Call `deleteAlarm()` explicitly before `deleteAll()` to remove alarm

### "Migration Rollback Not Supported"

**Cause:** Attempting to rollback a migration after deployment
**Solution:** Test with `--dry-run` before deploying; migrations cannot be rolled back

### "deleted_classes Destroys Data"

**Problem:** Migration deleted all data
**Cause:** `deleted_classes` migration immediately destroys all DO instances and data
**Solution:** Test with `--dry-run`; use `transferred_classes` to preserve data during moves

### "Cold Starts Are Slow"

**Problem:** First request after eviction takes longer
**Cause:** DO constructor + initial storage access on cold start
**Solution:** Expected behavior; optimize constructor, use connection pooling in clients, consider warming strategy for critical DOs

```typescript
// Warming strategy (periodically ping critical DOs)
export default {
  async scheduled(event: ScheduledEvent, env: Env) {
    const criticalIds = ["auth", "sessions", "locks"];
    await Promise.all(criticalIds.map(name => {
      const id = env.MY_DO.idFromName(name);
      const stub = env.MY_DO.get(id);
      return stub.ping();  // Keep warm
    }));
  }
};
```

## Limits

| Limit | Free | Paid | Notes |
|-------|------|------|-------|
| SQLite storage per DO | 10 GB | 10 GB | Per Durable Object instance |
| SQLite total storage | 5 GB | Unlimited | Account-wide quota |
| Key+value size | 2 MB | 2 MB | Single KV pair (SQLite/async) |
| CPU time default | 30s | 30s | Per request; configurable |
| CPU time max | 300s | 300s | Set via `limits.cpu_ms` |
| DO classes | 100 | 500 | Distinct DO class definitions |
| SQL columns | 100 | 100 | Per table |
| SQL statement size | 100 KB | 100 KB | Max SQL query size |
| SQL parameters | 100 | 100 | Per query |
| LIKE/GLOB pattern | 50 B | 50 B | Max pattern size |
| WebSocket message size | 32 MiB | 32 MiB | Per message |
| Request throughput | ~1K req/s | ~1K req/s | Per DO (soft limit - shard for more) |
| Alarms per DO | 1 | 1 | Use queue pattern for multiple events |
| Total DOs | Unlimited | Unlimited | Create as many instances as needed |
| WebSockets | Unlimited | Unlimited | Within 128MB memory limit per DO |
| Memory per DO | 128 MB | 128 MB | In-memory state + WebSocket buffers |
| KV key size | 2 KiB | 2 KiB | KV-style storage |
| KV value size | 128 KiB | 128 KiB | KV-style storage |

## Hibernation Caveats

1. **Memory cleared** - All in-memory variables lost; reconstruct from storage or `deserializeAttachment()`
2. **Constructor reruns** - Runs on wake; avoid expensive operations, use lazy initialization
3. **No guarantees** - DO may evict instead of hibernate; design for both
4. **Attachment limit** - `serializeAttachment()` data must be JSON-serializable, keep small
5. **Alarm wakes DO** - Alarm prevents hibernation until handler completes
6. **WebSocket state not automatic** - Must explicitly persist with `serializeAttachment()` or storage

## See Also

- **[Patterns](./patterns.md)** - Workarounds for common limitations
- **[API](./api.md)** - Storage limits and quotas
- **[Configuration](./configuration.md)** - Setting CPU limits
