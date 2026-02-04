# Testing Durable Objects

Use `@cloudflare/vitest-pool-workers` to test DOs inside the Workers runtime.

## Setup

### Install Dependencies

```bash
npm i -D vitest@~3.2.0 @cloudflare/vitest-pool-workers
```

### vitest.config.ts

```typescript
import { defineWorkersConfig } from "@cloudflare/vitest-pool-workers/config";

export default defineWorkersConfig({
  test: {
    poolOptions: {
      workers: {
        wrangler: { configPath: "./wrangler.toml" },
      },
    },
  },
});
```

### TypeScript Config (test/tsconfig.json)

```jsonc
{
  "extends": "../tsconfig.json",
  "compilerOptions": {
    "moduleResolution": "bundler",
    "types": ["@cloudflare/vitest-pool-workers"]
  },
  "include": ["./**/*.ts", "../src/worker-configuration.d.ts"]
}
```

### Environment Types (env.d.ts)

```typescript
declare module "cloudflare:test" {
  interface ProvidedEnv extends Env {}
}
```

## Unit Tests (Direct DO Access)

```typescript
import { env } from "cloudflare:test";
import { describe, it, expect } from "vitest";

describe("Counter DO", () => {
  it("should increment", async () => {
    const stub = env.COUNTER.getByName("test-counter");

    expect(await stub.increment()).toBe(1);
    expect(await stub.increment()).toBe(2);
    expect(await stub.getCount()).toBe(2);
  });

  it("isolates different instances", async () => {
    const stub1 = env.COUNTER.getByName("counter-1");
    const stub2 = env.COUNTER.getByName("counter-2");

    await stub1.increment();
    await stub1.increment();
    await stub2.increment();

    expect(await stub1.getCount()).toBe(2);
    expect(await stub2.getCount()).toBe(1);
  });
});
```

## Integration Tests (HTTP via SELF)

```typescript
import { SELF } from "cloudflare:test";
import { describe, it, expect } from "vitest";

describe("Worker HTTP", () => {
  it("should increment via POST", async () => {
    const res = await SELF.fetch("http://example.com?id=test", {
      method: "POST",
    });

    expect(res.status).toBe(200);
    const data = await res.json<{ count: number }>();
    expect(data.count).toBe(1);
  });

  it("should get count via GET", async () => {
    await SELF.fetch("http://example.com?id=get-test", { method: "POST" });
    await SELF.fetch("http://example.com?id=get-test", { method: "POST" });

    const res = await SELF.fetch("http://example.com?id=get-test");
    const data = await res.json<{ count: number }>();
    expect(data.count).toBe(2);
  });
});
```

## Direct Internal Access

Use `runInDurableObject()` to access instance internals and storage:

```typescript
import { env, runInDurableObject } from "cloudflare:test";
import { describe, it, expect } from "vitest";
import { Counter } from "../src";

describe("DO internals", () => {
  it("can verify storage directly", async () => {
    const stub = env.COUNTER.getByName("direct-test");
    await stub.increment();
    await stub.increment();

    await runInDurableObject(stub, async (instance: Counter, state) => {
      expect(instance).toBeInstanceOf(Counter);

      const result = state.storage.sql
        .exec<{ value: number }>(
          "SELECT value FROM counters WHERE name = ?",
          "default"
        )
        .one();
      expect(result.value).toBe(2);
    });
  });
});
```

## List DO IDs

```typescript
import { env, listDurableObjectIds } from "cloudflare:test";
import { describe, it, expect } from "vitest";

describe("DO listing", () => {
  it("can list all IDs in namespace", async () => {
    const id1 = env.COUNTER.idFromName("list-1");
    const id2 = env.COUNTER.idFromName("list-2");

    await env.COUNTER.get(id1).increment();
    await env.COUNTER.get(id2).increment();

    const ids = await listDurableObjectIds(env.COUNTER);
    expect(ids.length).toBe(2);
    expect(ids.some(id => id.equals(id1))).toBe(true);
    expect(ids.some(id => id.equals(id2))).toBe(true);
  });
});
```

## Testing Alarms

Use `runDurableObjectAlarm()` to trigger alarms immediately:

```typescript
import { env, runInDurableObject, runDurableObjectAlarm } from "cloudflare:test";
import { describe, it, expect } from "vitest";

describe("DO alarms", () => {
  it("can trigger alarms immediately", async () => {
    const stub = env.COUNTER.getByName("alarm-test");
    await stub.increment();
    await stub.increment();
    expect(await stub.getCount()).toBe(2);

    // Schedule alarm
    await runInDurableObject(stub, async (instance, state) => {
      await state.storage.setAlarm(Date.now() + 60_000);
    });

    // Execute immediately without waiting
    const ran = await runDurableObjectAlarm(stub);
    expect(ran).toBe(true);

    // Verify alarm handler ran (if it resets counter)
    expect(await stub.getCount()).toBe(0);

    // No alarm scheduled now
    const ranAgain = await runDurableObjectAlarm(stub);
    expect(ranAgain).toBe(false);
  });
});
```

Example alarm handler:
```typescript
async alarm(): Promise<void> {
  this.ctx.storage.sql.exec("DELETE FROM counters");
}
```

## Test Isolation

Each test gets isolated storage automatically. DOs from one test don't affect others:

```typescript
describe("Isolation", () => {
  it("first test creates DO", async () => {
    const stub = env.COUNTER.getByName("isolated");
    await stub.increment();
    expect(await stub.getCount()).toBe(1);
  });

  it("second test has fresh state", async () => {
    const ids = await listDurableObjectIds(env.COUNTER);
    expect(ids.length).toBe(0); // Previous test's DO is gone

    const stub = env.COUNTER.getByName("isolated");
    expect(await stub.getCount()).toBe(0); // Fresh instance
  });
});
```

## SQLite Storage Testing

```typescript
describe("SQLite", () => {
  it("can verify SQL storage", async () => {
    const stub = env.COUNTER.getByName("sqlite-test");
    await stub.increment("page-views");
    await stub.increment("page-views");
    await stub.increment("api-calls");

    await runInDurableObject(stub, async (instance, state) => {
      const rows = state.storage.sql
        .exec<{ name: string; value: number }>(
          "SELECT name, value FROM counters ORDER BY name"
        )
        .toArray();

      expect(rows).toEqual([
        { name: "api-calls", value: 1 },
        { name: "page-views", value: 2 },
      ]);

      expect(state.storage.sql.databaseSize).toBeGreaterThan(0);
    });
  });
});
```

## Testing Concurrency

```typescript
it("handles concurrent increments safely", async () => {
  const id = env.COUNTER.idFromName("concurrent-test");

  // Parallel increments
  const results = await Promise.all([
    runInDurableObject(env.COUNTER, id, (i) => i.increment()),
    runInDurableObject(env.COUNTER, id, (i) => i.increment()),
    runInDurableObject(env.COUNTER, id, (i) => i.increment())
  ]);

  // All should get unique values
  expect(new Set(results).size).toBe(3);
  expect(Math.max(...results)).toBe(3);
});
```

## Testing PITR (Point-in-Time Recovery)

```typescript
it("restores from bookmark", async () => {
  const id = env.MY_DO.idFromName("pitr-test");

  // Create checkpoint
  const bookmark = await runInDurableObject(env.MY_DO, id, async (instance, state) => {
    await state.storage.put("value", 1);
    return await state.storage.getCurrentBookmark();
  });

  // Modify and restore
  await runInDurableObject(env.MY_DO, id, async (instance, state) => {
    await state.storage.put("value", 2);
    await state.storage.onNextSessionRestoreBookmark(bookmark);
    state.abort();
  });

  // Verify restored
  await runInDurableObject(env.MY_DO, id, async (instance, state) => {
    const value = await state.storage.get("value");
    expect(value).toBe(1);
  });
});
```

## Testing Transactions

```typescript
it("rolls back on error", async () => {
  const id = env.BANK.idFromName("transaction-test");

  await runInDurableObject(env.BANK, id, async (instance, state) => {
    await state.storage.put("balance", 100);

    await expect(
      state.storage.transaction(async () => {
        await state.storage.put("balance", 50);
        throw new Error("Cancel");
      })
    ).rejects.toThrow("Cancel");

    const balance = await state.storage.get("balance");
    expect(balance).toBe(100); // Rolled back
  });
});
```

## Per-Test Isolation Patterns

```typescript
// Per-test unique IDs
let testId: string;
beforeEach(() => { testId = crypto.randomUUID(); });

it("isolated test", async () => {
  const id = env.MY_DO.idFromName(testId);
  // Uses unique DO instance
});

// Cleanup pattern
it("with cleanup", async () => {
  const id = env.MY_DO.idFromName("cleanup-test");
  try {
    await runInDurableObject(env.MY_DO, id, async (instance) => {});
  } finally {
    await runInDurableObject(env.MY_DO, id, async (instance, state) => {
      await state.storage.deleteAll();
    });
  }
});
```

## Running Tests

```bash
npx vitest        # Watch mode
npx vitest run    # Single run
```

package.json:
```json
{
  "scripts": {
    "test": "vitest"
  }
}
```

## See Also

- **[API](./api.md)** - Alarm and storage APIs
- **[Patterns](./patterns.md)** - Common DO patterns to test
- **[Gotchas](./gotchas.md)** - Common testing issues
