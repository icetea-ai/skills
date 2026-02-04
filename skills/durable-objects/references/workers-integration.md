# Workers Integration

Best practices for Workers that invoke Durable Objects.

## Wrangler Configuration

### wrangler.jsonc (Recommended)

```jsonc
{
  "$schema": "node_modules/wrangler/config-schema.json",
  "name": "my-worker",
  "main": "src/index.ts",
  "compatibility_date": "2024-12-01",
  "compatibility_flags": ["nodejs_compat"],

  "durable_objects": {
    "bindings": [
      { "name": "CHAT_ROOM", "class_name": "ChatRoom" },
      { "name": "USER_SESSION", "class_name": "UserSession" }
    ]
  },

  "migrations": [
    { "tag": "v1", "new_sqlite_classes": ["ChatRoom", "UserSession"] }
  ],

  // Environment variables
  "vars": {
    "ENVIRONMENT": "production"
  },

  // KV namespaces
  "kv_namespaces": [
    { "binding": "CONFIG", "id": "abc123" }
  ],

  // R2 buckets
  "r2_buckets": [
    { "binding": "UPLOADS", "bucket_name": "my-uploads" }
  ],

  // D1 databases
  "d1_databases": [
    { "binding": "DB", "database_id": "xyz789" }
  ]
}
```

### wrangler.toml (Alternative)

```toml
name = "my-worker"
main = "src/index.ts"
compatibility_date = "2024-12-01"
compatibility_flags = ["nodejs_compat"]

[[durable_objects.bindings]]
name = "CHAT_ROOM"
class_name = "ChatRoom"

[[migrations]]
tag = "v1"
new_sqlite_classes = ["ChatRoom"]

[vars]
ENVIRONMENT = "production"
```

## TypeScript Types

### Environment Interface

```typescript
// src/types.ts
import { ChatRoom } from "./durable-objects/chat-room";
import { UserSession } from "./durable-objects/user-session";

export interface Env {
  // Durable Objects
  CHAT_ROOM: DurableObjectNamespace<ChatRoom>;
  USER_SESSION: DurableObjectNamespace<UserSession>;

  // KV
  CONFIG: KVNamespace;

  // R2
  UPLOADS: R2Bucket;

  // D1
  DB: D1Database;

  // Environment variables
  ENVIRONMENT: string;
  API_KEY: string; // From secrets
}
```

### Export Durable Object Classes

```typescript
// src/index.ts
export { ChatRoom } from "./durable-objects/chat-room";
export { UserSession } from "./durable-objects/user-session";

export default {
  async fetch(request: Request, env: Env, ctx: ExecutionContext): Promise<Response> {
    // Worker handler
  },
};
```

## Worker Handler Pattern

```typescript
export default {
  async fetch(request: Request, env: Env, ctx: ExecutionContext): Promise<Response> {
    const url = new URL(request.url);

    try {
      // Route to appropriate handler
      if (url.pathname.startsWith("/api/rooms")) {
        return handleRooms(request, env);
      }
      if (url.pathname.startsWith("/api/users")) {
        return handleUsers(request, env);
      }

      return new Response("Not Found", { status: 404 });
    } catch (error) {
      console.error("Request failed:", error);
      return new Response("Internal Server Error", { status: 500 });
    }
  },
};

async function handleRooms(request: Request, env: Env): Promise<Response> {
  const url = new URL(request.url);
  const roomId = url.searchParams.get("room");

  if (!roomId) {
    return Response.json({ error: "Missing room parameter" }, { status: 400 });
  }

  const stub = env.CHAT_ROOM.getByName(roomId);

  if (request.method === "POST") {
    const body = await request.json<{ userId: string; message: string }>();
    const result = await stub.sendMessage(body.userId, body.message);
    return Response.json(result);
  }

  const messages = await stub.getMessages();
  return Response.json(messages);
}
```

## Request Validation with Zod

```typescript
import { z } from "zod";

const SendMessageSchema = z.object({
  userId: z.string().min(1),
  message: z.string().min(1).max(1000),
});

async function handleSendMessage(request: Request, env: Env): Promise<Response> {
  const body = await request.json();
  const result = SendMessageSchema.safeParse(body);

  if (!result.success) {
    return Response.json(
      { error: "Validation failed", details: result.error.issues },
      { status: 400 }
    );
  }

  const stub = env.CHAT_ROOM.getByName(result.data.userId);
  const message = await stub.sendMessage(result.data.userId, result.data.message);
  return Response.json(message);
}
```

## DO ID Strategy and Security

How you obtain DO IDs affects your security posture:

### idFromName() - Requires Validation

**Any string creates a valid ID.** Validate before routing to prevent "leaked DOs":

```typescript
// WRONG: Accepts any user input
async function handleRoom(request: Request, env: Env): Promise<Response> {
  const url = new URL(request.url);
  const roomId = url.searchParams.get("room");
  const stub = env.CHAT_ROOM.getByName(roomId);  // Creates DO for ANY string!
  return stub.handleRequest(request);
}

// RIGHT: Validate existence/authorization first
async function handleRoom(request: Request, env: Env): Promise<Response> {
  const url = new URL(request.url);
  const roomId = url.searchParams.get("room");

  const room = await env.DB.prepare(
    "SELECT 1 FROM rooms WHERE id = ? AND is_active = true"
  ).bind(roomId).first();

  if (!room) {
    return Response.json({ error: "Room not found" }, { status: 404 });
  }

  const stub = env.CHAT_ROOM.getByName(roomId);  // Safe - room exists
  return stub.handleRequest(request);
}
```

### idFromString() - Built-in Protection

**Fails for invalid IDs.** Safe to use with untrusted input:

```typescript
async function handleVoucher(request: Request, env: Env): Promise<Response> {
  const url = new URL(request.url);
  const idString = url.searchParams.get("id");

  try {
    const id = env.VOUCHER.idFromString(idString);  // Throws if invalid!
    const stub = env.VOUCHER.get(id);
    return stub.handleRequest(request);
  } catch {
    return Response.json({ error: "Invalid voucher ID" }, { status: 400 });
  }
}
```

### Factory Pattern - Best for Human-Readable Identifiers

Separate user-facing codes from internal DO IDs:

```typescript
// User provides: "SAVE20" (human-readable voucher code)
// System uses: "abc123..." (64-char DO ID)

async function handleRedeem(request: Request, env: Env): Promise<Response> {
  const url = new URL(request.url);
  const voucherCode = url.searchParams.get("code");  // "SAVE20"

  // Lookup: human code -> DO ID
  const voucher = await env.DB.prepare(
    "SELECT do_id FROM vouchers WHERE code = ? AND active = true"
  ).bind(voucherCode).first();

  if (!voucher) {
    return Response.json({ error: "Invalid voucher" }, { status: 404 });
  }

  const id = env.VOUCHER.idFromString(voucher.do_id);
  const stub = env.VOUCHER.get(id);
  return stub.redeem(request);
}
```

This pattern ensures:
- Users never see/guess DO IDs
- Only pre-created DOs can be accessed
- Clean separation of concerns

## Observability & Logging

### Structured Logging

```typescript
function log(level: "info" | "warn" | "error", message: string, data?: Record<string, unknown>) {
  console.log(JSON.stringify({
    level,
    message,
    timestamp: new Date().toISOString(),
    ...data,
  }));
}

// Usage
log("info", "Request received", { path: url.pathname, method: request.method });
log("error", "DO call failed", { roomId, error: String(error) });
```

### Request Tracing

```typescript
async function handleRequest(request: Request, env: Env): Promise<Response> {
  const requestId = crypto.randomUUID();
  const startTime = Date.now();

  try {
    const response = await processRequest(request, env);

    log("info", "Request completed", {
      requestId,
      duration: Date.now() - startTime,
      status: response.status,
    });

    return response;
  } catch (error) {
    log("error", "Request failed", {
      requestId,
      duration: Date.now() - startTime,
      error: String(error),
    });
    throw error;
  }
}
```

### Tail Workers (Production)

For production logging, use Tail Workers to forward logs:

```jsonc
// wrangler.jsonc
{
  "tail_consumers": [
    { "service": "log-collector" }
  ]
}
```

## Error Handling

### Graceful DO Errors

```typescript
async function callDO(stub: DurableObjectStub<ChatRoom>, method: string): Promise<Response> {
  try {
    const result = await stub.getMessages();
    return Response.json(result);
  } catch (error) {
    if (error instanceof Error) {
      // DO threw an error
      log("error", "DO operation failed", { error: error.message });
      return Response.json(
        { error: "Service temporarily unavailable" },
        { status: 503 }
      );
    }
    throw error;
  }
}
```

### Timeout Handling

```typescript
async function withTimeout<T>(promise: Promise<T>, ms: number): Promise<T> {
  const timeout = new Promise<never>((_, reject) =>
    setTimeout(() => reject(new Error("Timeout")), ms)
  );
  return Promise.race([promise, timeout]);
}

// Usage
const result = await withTimeout(stub.processData(data), 5000);
```

## CORS Handling

```typescript
function corsHeaders(): HeadersInit {
  return {
    "Access-Control-Allow-Origin": "*",
    "Access-Control-Allow-Methods": "GET, POST, PUT, DELETE, OPTIONS",
    "Access-Control-Allow-Headers": "Content-Type, Authorization",
  };
}

export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    if (request.method === "OPTIONS") {
      return new Response(null, { headers: corsHeaders() });
    }

    const response = await handleRequest(request, env);

    // Add CORS headers to response
    const newHeaders = new Headers(response.headers);
    Object.entries(corsHeaders()).forEach(([k, v]) => newHeaders.set(k, v));

    return new Response(response.body, {
      status: response.status,
      headers: newHeaders,
    });
  },
};
```

## Secrets Management

Set secrets via wrangler CLI (not in config files):

```bash
wrangler secret put API_KEY
wrangler secret put DATABASE_URL
```

Access in code:
```typescript
export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const apiKey = env.API_KEY; // From secret
    // ...
  },
};
```

## Development Commands

```bash
# Local development
wrangler dev

# Deploy
wrangler deploy

# Tail logs
wrangler tail

# List DOs
wrangler d1 execute DB --command "SELECT * FROM _cf_DO"
```

## See Also

- **[Configuration](./configuration.md)** - Detailed wrangler setup
- **[API](./api.md)** - DO class structure and methods
- **[Testing](./testing.md)** - Testing Workers with DOs
