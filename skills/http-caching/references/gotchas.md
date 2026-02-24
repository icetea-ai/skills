# HTTP Caching Gotchas

## `no-cache` Does NOT Mean "Don't Cache"

`no-cache` means "store but always revalidate before using." The response IS written to cache.

| Directive | Stored? | Served without revalidation? |
|---|---|---|
| `no-cache` | Yes | No (must revalidate every time) |
| `no-store` | No | N/A — never stored |

**Why store if you always revalidate?** Because the browser keeps a local copy. When it revalidates and the content hasn't changed, the server responds with a `304 Not Modified` (~200 bytes, headers only) instead of re-sending the full body (potentially 50KB+). Storing enables this bandwidth savings.

**If you want to prevent caching entirely, use `no-store`.** Use `no-cache` when you want guaranteed freshness with efficient revalidation.

## Kitchen-Sink Anti-Pattern

```
Cache-Control: no-store, no-cache, max-age=0, must-revalidate, private
```

This is contradictory and wasteful. `no-store` already prevents caching — the other directives are meaningless alongside it. `no-cache` + `must-revalidate` are redundant with each other.

**Pick ONE strategy:**
- Never cache → `no-store`
- Always revalidate → `no-cache` (+ ETag for efficiency)
- Cache with TTL → `max-age=N` (+ `must-revalidate` if serving stale is unacceptable)

## Heuristic Caching Risk

**No `Cache-Control` header does NOT disable caching.**

When a response has no `Cache-Control` and no `Expires`, browsers apply **heuristic caching**: they cache the response for ~10% of the time since `Last-Modified`. Per [MDN](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/Caching#heuristic_caching): *"the specification recommends about 10% of the time after storing."*

Example: `Last-Modified: 90 days ago` → browser may cache for ~9 days with no way to revalidate.

For simple static sites, heuristic caching is often fine. But set explicit `Cache-Control` headers when you need **predictable behavior** — especially for APIs, authenticated content, or versioned assets where staleness causes real problems.

## `Vary: User-Agent` Cache Explosion

`Vary: User-Agent` creates a separate cache entry for every unique User-Agent string. With thousands of browser/OS/version combinations, your cache hit rate drops to near zero.

**Instead of `Vary: User-Agent`:**
- Serve the same response to all clients (responsive design)
- If you must vary, use `Vary: Accept` or a custom header with low cardinality (e.g., `X-Device-Class: mobile|desktop`)

## Missing `Vary: Origin` on CORS Endpoints

Without `Vary: Origin` on responses that reflect the requesting origin:
1. CDN caches response with `Access-Control-Allow-Origin: https://a.com`
2. Request from `https://b.com` gets the cached response
3. Browser sees wrong origin → blocks the request

**Always include `Vary: Origin`** when using origin-reflecting CORS (anything other than `Access-Control-Allow-Origin: *`).

## Browser Caps on `Access-Control-Max-Age`

| Browser | Max Value |
|---|---|
| Chrome | 7200s (2 hours) |
| Firefox | 86400s (24 hours) |
| Default (header omitted) | 5 seconds |

Setting `Access-Control-Max-Age: 86400` still means Chrome re-sends preflights every 2 hours. The 5-second default when the header is omitted means a preflight for nearly every CORS request.

## Revalidation Is Efficient, Not Pessimistic

`no-cache` + ETag is NOT a slow strategy. A `304 Not Modified` response is headers-only (~200 bytes). Compare:
- Full response: 50KB+ body transfer
- 304 revalidation: ~200 bytes (headers only)

For content that changes infrequently, `no-cache` + ETag gives you guaranteed freshness with minimal bandwidth. This is the right default for HTML pages and API responses where staleness is unacceptable.

## `immutable` Still Needs `max-age`

`immutable` suppresses browser revalidation (even on Ctrl+R reload), but only **during the freshness window** defined by `max-age`. It does NOT define how long to cache — without `max-age`, there's no freshness window for `immutable` to apply to.

```
# WRONG: immutable alone — no defined freshness lifetime
Cache-Control: public, immutable

# CORRECT: max-age defines the window, immutable suppresses revalidation within it
Cache-Control: public, max-age=31536000, immutable
```

**Why both?** `immutable` isn't universally supported. `max-age=31536000` ensures all caches (including those that don't understand `immutable`) still cache the resource for a year. CDNs ignore `immutable` entirely — they follow `max-age`/`s-maxage` as usual.

For the full hashed-filename strategy that makes this safe, see "Static Versioned Assets" in [patterns.md](./patterns.md).

## `s-maxage` Without `public`

Some CDNs require `public` to be present before they'll cache a response, even if `s-maxage` is set. While the HTTP spec says `s-maxage` implies public cacheability, be explicit:
```
Cache-Control: public, s-maxage=3600, max-age=60
```
