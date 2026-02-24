# HTTP Cache Directives Reference

## Response Directives (Cache-Control)

| Directive | Applies To | Meaning |
|---|---|---|
| `max-age=N` | Browser + CDN | Cache is fresh for N seconds from response time |
| `s-maxage=N` | CDN/shared only | Overrides `max-age` for shared caches (CDN, proxy). Browser ignores this. |
| `no-cache` | Browser + CDN | **Store but always revalidate before using.** Does NOT mean "don't cache." |
| `no-store` | Browser + CDN | Never store. Response must be fetched fresh every time. |
| `private` | Browser only | Only browser may cache. CDN/proxies must not store. |
| `public` | Browser + CDN | Any cache may store. Required for CDN caching of responses with `Authorization` header. |
| `must-revalidate` | Browser + CDN | Once stale, cache MUST revalidate with origin before serving. If origin is unreachable, returns **504 Gateway Timeout** — never serves stale. Request blocks until origin responds. |
| `proxy-revalidate` | CDN/shared only | Like `must-revalidate` but only for shared caches. |
| `immutable` | Browser | Browser skips revalidation even on reload, but only during the `max-age` freshness window. Does NOT define cache lifetime — `max-age` is still required. |
| `stale-while-revalidate=N` | Browser + CDN | Serve stale for N seconds while revalidating in background. |
| `stale-if-error=N` | Browser + CDN | Serve stale for N seconds if origin returns 5xx or is unreachable. |
| `no-transform` | CDN/proxy | Proxies must not modify response body (no re-compression, image optimization). |

## Request Directives (Cache-Control)

Sent by the client to override cache behavior:

| Directive | Meaning |
|---|---|
| `max-age=0` | Force revalidation (equivalent to browser hard-reload behavior) |
| `no-cache` | Same as `max-age=0` — revalidate before serving |
| `no-store` | Do not use any cached response |
| `only-if-cached` | Return cached response only — fail with 504 if not cached |
| `max-stale=N` | Accept stale response up to N seconds past expiry |
| `min-fresh=N` | Response must remain fresh for at least N more seconds |

## Validation Headers

### ETag (Entity Tag)

Server-generated identifier for a specific version of a resource.

| Type | Format | Comparison | Use Case |
|---|---|---|---|
| Strong | `"abc123"` | Byte-for-byte identical | Default. Use for most resources. |
| Weak | `W/"abc123"` | Semantically equivalent | When byte-level identity isn't needed (e.g., cosmetic HTML changes). |

### Last-Modified

Timestamp of last change. Less precise than ETag (1-second granularity). Use ETag when possible.

```
Last-Modified: Tue, 15 Nov 2024 12:45:26 GMT
```

### Conditional Request Headers

| Request Header | Paired With | Server Response |
|---|---|---|
| `If-None-Match: "abc123"` | `ETag` | `304 Not Modified` if ETag matches, else `200` with new body |
| `If-Modified-Since: <date>` | `Last-Modified` | `304 Not Modified` if unchanged, else `200` with new body |

`If-None-Match` takes precedence over `If-Modified-Since` when both are present.

## Vary Header

Tells caches that the response depends on specific request headers. Each unique combination of `Vary` header values creates a **separate cache entry**.

```
Vary: Accept-Encoding, Origin
```

| Value | When to Use |
|---|---|
| `Accept-Encoding` | Serving compressed responses (gzip/brotli). Most CDNs handle automatically. |
| `Origin` | CORS responses that reflect the requesting origin. **Required** to prevent cache poisoning. |
| `Accept` | Content negotiation (JSON vs XML). |
| `Cookie` | User-specific responses cached at CDN. Use sparingly — high cardinality. |
| `Authorization` | Rarely useful — prefer `private` or `no-store` instead. |

**Avoid `Vary: User-Agent`** — thousands of unique values = thousands of cache entries = cache explosion with near-zero hit rate.

## Precedence Rules

1. `no-store` wins over everything — response is never cached
2. `no-cache` requires revalidation regardless of freshness
3. `s-maxage` overrides `max-age` for shared caches (CDN/proxy) only
4. `must-revalidate` makes stale requests **blocking** — cache waits for origin, returns 504 if unreachable. Without it, caches MAY serve stale when origin is down. Overrides `stale-while-revalidate` after its window.
5. `private` prevents shared caching (CDN ignores the response)
6. `immutable` only affects browser revalidation behavior — CDN ignores it

When directives conflict, the most restrictive interpretation applies.
