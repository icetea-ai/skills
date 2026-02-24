# HTTP Caching Patterns

## Static Versioned Assets (Fire-and-Forget)

**The pattern:** Content-hash in filename → cache forever → never revalidate.

```
/assets/main.a1b2c3d4.js
/assets/styles.e5f6g7h8.css
/assets/logo.i9j0k1l2.png
```

Headers:
```
Cache-Control: public, max-age=31536000, immutable
```

**Why this works:** The URL is the version. When content changes, the URL changes. Old URLs become unreachable via new HTML, so stale content is impossible. No server logic needed — just serve forever.

**`immutable` matters:** Without it, browsers send conditional requests on page reload even when `max-age` hasn't expired. `immutable` eliminates these round-trips entirely.

**Requirements:**
- Build tool must produce hashed filenames (webpack, Vite, esbuild all do this)
- HTML that references these assets must NOT be cached long-term (it needs to point to new hashes)

## HTML and Dynamic Pages

**The pattern:** Always revalidate, but minimize transfer with conditional requests.

```
Cache-Control: no-cache
ETag: "page-v42"
```

On subsequent requests, the browser sends `If-None-Match: "page-v42"`. If unchanged → `304 Not Modified` (headers only, ~200 bytes). If changed → `200` with full body.

**Why `no-cache`, not `no-store`:** The page IS cached (saved to disk), just always validated before use. This means the browser has a local copy ready — revalidation is fast (headers-only 304), and if the server is unreachable with `stale-if-error`, the browser could serve the cached copy.

**ETag requires server-side support.** The server must both generate ETags and handle `If-None-Match` comparison to return 304. Sending an ETag without implementing the comparison logic means the server always returns 200 with the full body — the ETag is wasted. Many frameworks handle this automatically (Express `etag` middleware, Nginx for static files, most CDNs). For custom APIs, you must implement the comparison yourself.

For authenticated pages, see the [Authenticated API](#authenticated-api) pattern below — same headers apply.

## URL Consistency Principle

**Different URLs = different cache entries.** Treat cache keys as a first-class concern.

Problems:
- `/api/users` vs `/api/users/` — two cache entries for same data
- Query parameter ordering: `?a=1&b=2` vs `?b=2&a=1` — two entries
- Trailing slashes, case differences, redundant params all fragment the cache

Solutions:
- Normalize URLs at CDN edge (redirect or rewrite)
- Sort query parameters
- Enforce consistent URL formats in your application

## API Response Caching

### Public API (no auth)

```
Cache-Control: public, s-maxage=3600, max-age=60
ETag: "data-v15"
Vary: Accept-Encoding
```

- `s-maxage=3600`: CDN serves cached response for 1 hour
- `max-age=60`: Browser caches for 1 minute (gets fresher data on revisit)
- ETag enables efficient revalidation at both layers

### Authenticated API

```
Cache-Control: private, no-cache
ETag: "user42-data-v3"
```

- `private`: CDN must not cache (response is user-specific)
- `no-cache`: Browser always revalidates (data may change between requests)
- ETag: Efficient 304 responses when data hasn't changed

## CDN Layering (Browser vs CDN TTLs)

`s-maxage` controls CDN TTL independently from browser `max-age`:

```
Cache-Control: public, s-maxage=86400, max-age=300
```

- CDN caches for 24 hours (reduces origin load)
- Browser caches for 5 minutes (gets reasonably fresh data)
- CDN-to-origin revalidation uses ETag (304s save bandwidth)

**When to split TTLs:**
- Origin is expensive to hit → long `s-maxage`
- Content changes moderately → short `max-age` for browsers
- CDN supports purging → long `s-maxage` + purge on change

## Stale-While-Revalidate (SWR)

```
Cache-Control: public, max-age=60, stale-while-revalidate=3600
```

**Flow:**
1. First 60s: Serve from cache (fresh)
2. 60s–3660s: Serve stale immediately, revalidate in background
3. After 3660s: Cache entry is fully stale, must fetch fresh

**UX benefit:** Users never wait for revalidation. They always get an instant response (potentially stale by seconds/minutes).

**Pair with `stale-if-error`** for resilience:
```
Cache-Control: public, max-age=60, stale-while-revalidate=3600, stale-if-error=86400
```

## ISR (Incremental Static Regeneration)

Frameworks like Next.js use HTTP caching primitives for ISR:

```
Cache-Control: public, s-maxage=3600, stale-while-revalidate=86400
```

**How it works:**
1. CDN serves cached HTML for up to 1 hour (fresh)
2. After 1 hour, CDN serves stale while triggering background regeneration
3. Framework rebuilds the page, CDN stores new version
4. Next request gets fresh page

**Cache tags for selective purge:** Many CDNs support `Surrogate-Key` or `Cache-Tag` headers for targeted invalidation:
```
Surrogate-Key: product-123 category-electronics
Cache-Tag: product-123, category-electronics
```
Purge by tag when data changes instead of waiting for TTL expiry.

## CORS Response Caching

Three separate caching concerns that must work together:

### 1. Browser Preflight Cache (`Access-Control-Max-Age`)

```
Access-Control-Max-Age: 7200
```

Tells the browser to cache the preflight (OPTIONS) result. Browser-specific caps:
- Chrome: **7200s** (2 hours)
- Firefox: **86400s** (24 hours)
- Default when header is omitted: **5 seconds**

Setting values above browser caps is harmless but ineffective for that browser.

### 2. CDN Preflight Cache

Most CDNs do NOT cache OPTIONS by default. You must explicitly configure:
```
Cache-Control: public, max-age=86400
```
on OPTIONS responses, plus a CDN rule to cache the OPTIONS method.

### 3. `Vary: Origin` on ALL responses (OPTIONS and actual)

```
Vary: Origin
Access-Control-Allow-Origin: https://app.example.com
```

**Required whenever you reflect the requesting `Origin` back** (anything other than `Access-Control-Allow-Origin: *`). Without `Vary: Origin`:
- CDN caches response for Origin A
- Request from Origin B gets Origin A's CORS headers from cache
- Browser blocks the request → **cache poisoning**

**Complete preflight response:**
```
HTTP/1.1 204 No Content
Access-Control-Allow-Origin: https://app.example.com
Access-Control-Allow-Methods: GET, POST, PUT, DELETE
Access-Control-Allow-Headers: Authorization, Content-Type
Access-Control-Max-Age: 7200
Cache-Control: public, max-age=7200
Vary: Origin, Access-Control-Request-Method, Access-Control-Request-Headers
```

**Complete actual response:**
```
HTTP/1.1 200 OK
Access-Control-Allow-Origin: https://app.example.com
Cache-Control: public, s-maxage=300, max-age=60
Vary: Origin, Accept-Encoding
ETag: "data-v1"
```

## Example: Cloudflare Workers Caching Headers

```ts
export default {
  async fetch(request, env) {
    const url = new URL(request.url);

    // Versioned static assets (hashed filenames) — cache forever
    if (url.pathname.startsWith("/assets/")) {
      const asset = await env.ASSETS.fetch(request);
      return new Response(asset.body, {
        headers: {
          ...Object.fromEntries(asset.headers),
          "Cache-Control": "public, max-age=31536000, immutable",
        },
      });
    }

    // API response — CDN caches 5 min, browser 1 min, ETag for revalidation
    if (url.pathname.startsWith("/api/")) {
      const data = await getApiData(url);
      const etag = `"${await hashContent(data)}"`;
      if (request.headers.get("If-None-Match") === etag) {
        return new Response(null, { status: 304 });
      }
      return Response.json(data, {
        headers: {
          "Cache-Control": "public, s-maxage=300, max-age=60",
          "ETag": etag,
        },
      });
    }

    // HTML — always revalidate
    const page = await renderPage(url);
    return new Response(page, {
      headers: {
        "Content-Type": "text/html",
        "Cache-Control": "no-cache",
        "ETag": `"${await hashContent(page)}"`,
      },
    });
  },
} satisfies ExportedHandler<Env>;
```

## Cache Invalidation Strategies

| Strategy | How | Best For |
|---|---|---|
| URL versioning | Hash in filename | Static assets (build-time) |
| Cache tags / surrogate keys | `Surrogate-Key` header + tag-based purge API | CMS content, e-commerce products |
| Event-driven purge | Webhook → CDN purge API on data change | Database-backed content |
| Time-based expiry | `max-age` / `s-maxage` | Content with known update frequency |
| Stale-while-revalidate | Background refresh on stale hit | Content where slight staleness is acceptable |

**Prefer URL versioning** for static assets — it's the only strategy that guarantees zero stale content without purge infrastructure.
