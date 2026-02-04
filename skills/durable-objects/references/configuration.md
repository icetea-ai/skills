# Durable Objects Configuration

## Basic Setup

```jsonc
{
  "name": "my-worker",
  "main": "src/index.ts",
  "compatibility_date": "2025-01-01",  // Use latest; >=2024-04-03 for RPC
  "durable_objects": {
    "bindings": [
      {
        "name": "MY_DO",                // Env binding name
        "class_name": "MyDO"            // Class exported from this worker
      },
      {
        "name": "EXTERNAL",             // Access DO from another worker
        "class_name": "ExternalDO",
        "script_name": "other-worker"
      }
    ]
  },
  "migrations": [
    { "tag": "v1", "new_sqlite_classes": ["MyDO"] }  // Prefer SQLite
  ]
}
```

## Binding Options

```jsonc
{
  "name": "BINDING_NAME",
  "class_name": "ClassName",
  "script_name": "other-worker",        // Optional: external DO
  "environment": "production"           // Optional: isolate by env
}
```

## Migrations

```jsonc
{
  "migrations": [
    { "tag": "v1", "new_sqlite_classes": ["MyDO"] },            // Create SQLite (recommended)
    // { "tag": "v1", "new_classes": ["MyDO"] },                // Create KV (paid only)
    { "tag": "v2", "renamed_classes": [{ "from": "Old", "to": "New" }] },
    { "tag": "v3", "transferred_classes": [{ "from": "Src", "from_script": "old", "to": "Dest" }] },
    { "tag": "v4", "deleted_classes": ["Obsolete"] }           // Destroys ALL data!
  ]
}
```

**Migration rules:**
- Tags must be unique and sequential (v1, v2, v3...)
- No rollback supported (test with `--dry-run` first)
- Auto-applied on deploy
- `new_sqlite_classes` recommended over `new_classes` (SQLite vs KV)
- `deleted_classes` immediately destroys ALL data (irreversible)

**Migration lifecycle:** Migrations run once per deployment. Existing DO instances get new storage backend on next invocation. Renaming/removing classes requires `renamed_classes` or `deleted_classes` entries.

## Storage Backend Selection

### SQLite-backed (Recommended)

```jsonc
{
  "migrations": [
    {
      "tag": "v1",
      "new_sqlite_classes": ["Counter", "Session", "RateLimiter"]
    }
  ]
}
```

### KV-backed (Legacy)

```jsonc
{
  "migrations": [
    {
      "tag": "v1",
      "new_classes": ["OldCounter"]
    }
  ]
}
```

## TypeScript Setup

```typescript
import { DurableObject } from "cloudflare:workers";

export class MyDurableObject extends DurableObject<Env> {
  constructor(ctx: DurableObjectState, env: Env) {
    super(ctx, env);

    // Initialize schema in constructor
    this.ctx.storage.sql.exec(`
      CREATE TABLE IF NOT EXISTS users(
        id INTEGER PRIMARY KEY,
        name TEXT NOT NULL,
        email TEXT UNIQUE
      );
    `);
  }
}

// Binding types
interface Env {
  MY_DO: DurableObjectNamespace<MyDurableObject>;
}

type DurableObjectNamespace<T> = {
  newUniqueId(options?: { jurisdiction?: string }): DurableObjectId;
  idFromName(name: string): DurableObjectId;
  idFromString(id: string): DurableObjectId;
  get(id: DurableObjectId): DurableObjectStub<T>;
};

export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const id = env.MY_DO.idFromName('singleton');
    const stub = env.MY_DO.get(id);

    // Modern RPC: call methods directly (recommended)
    const result = await stub.someMethod();
    return Response.json(result);

    // Legacy: forward request (still works)
    // return stub.fetch(request);
  }
}
```

## Jurisdiction (Data Locality)

Specify jurisdiction at ID creation for data residency compliance:

```typescript
// EU data residency
const id = env.MY_DO.idFromName("user:123", { jurisdiction: "eu" })

// Available jurisdictions
const jurisdictions = ["eu", "fedramp"]  // More may be added

// All operations on this DO stay within jurisdiction
const stub = env.MY_DO.get(id)
await stub.someMethod()  // Data stays in EU
```

**Key points:**
- Set at ID creation time, immutable afterward
- DO instance physically located within jurisdiction
- Storage and compute guaranteed within boundary
- Use for GDPR, FedRAMP, other compliance requirements
- No cross-jurisdiction access (requests fail if DO in different jurisdiction)

## Location Hints

```typescript
// Jurisdiction (GDPR/FedRAMP) - strict
const euNamespace = env.MY_DO.jurisdiction("eu");
const id = euNamespace.newUniqueId();
const stub = euNamespace.get(id);

// Location hint (best effort) - not strict
const stub = env.MY_DO.get(id, { locationHint: "enam" });
// Hints: wnam, enam, sam, weur, eeur, apac, oc, afr, me
```

## Environment Isolation

Separate DO namespaces per environment (staging/production have distinct object instances):

```jsonc
{
  "durable_objects": {
    "bindings": [{ "name": "MY_DO", "class_name": "MyDO" }]
  },
  "env": {
    "production": {
      "durable_objects": {
        "bindings": [
          { "name": "MY_DO", "class_name": "MyDO", "environment": "production" }
        ]
      }
    }
  }
}
```

Deploy: `npx wrangler deploy --env production`

## Initialization Pattern

```typescript
export class Counter extends DurableObject<Env> {
  value: number;

  constructor(ctx: DurableObjectState, env: Env) {
    super(ctx, env);

    // Block concurrent requests during init
    ctx.blockConcurrencyWhile(async () => {
      this.value = (await ctx.storage.get("value")) || 0;
    });
  }
}
```

## Limits & Settings

```jsonc
{
  "limits": {
    "cpu_ms": 300000  // Max CPU time: 30s default, 300s max
  }
}
```

See [Gotchas](./gotchas.md) for complete limits table.

## Commands

```bash
# Development
npx wrangler dev                    # Local dev
npx wrangler dev --remote           # Test against production DOs

# Deployment
npx wrangler deploy                 # Deploy + auto-apply migrations
npx wrangler deploy --dry-run       # Validate migrations without deploying
npx wrangler deploy --env production

# Management
npx wrangler durable-objects list                      # List namespaces
npx wrangler durable-objects info <namespace> <id>     # Inspect specific DO
npx wrangler durable-objects delete <namespace> <id>   # Delete DO (destroys data)
```

## See Also

- **[API](./api.md)** - DurableObjectState and lifecycle handlers
- **[Patterns](./patterns.md)** - Multi-environment patterns
- **[Gotchas](./gotchas.md)** - Migration caveats, limits
