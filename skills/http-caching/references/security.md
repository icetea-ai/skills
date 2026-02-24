# HTTP Caching Security

## Missing `private` on Authenticated Responses

The danger is specifically when you **explicitly enable caching** on authenticated responses without `private`:

```
# DANGEROUS: CDN caches user-specific data and serves it to everyone
Cache-Control: max-age=300
Cache-Control: public, max-age=300
```

The CDN serves User A's data to User B. Omitting `Cache-Control` entirely is a different problem — heuristic caching (see [gotchas.md](./gotchas.md)) — but most CDNs won't cache responses without explicit directives.

**Always be explicit.** Choose based on sensitivity:

```
# Sensitive data (tokens, PII) — never persist in any cache
Cache-Control: private, no-store
```

```
# User-specific data (profiles, dashboards) — browser caches, always revalidates
Cache-Control: private, no-cache
ETag: "user42-v3"
```

Use `no-store` when the data must never be written to disk. Use `no-cache` + ETag when you want efficient revalidation (304 responses save bandwidth).

## Account Switching Leaks

`private` prevents CDN caching but not cross-user issues within the same browser:

1. User A logs in → browser caches their profile with `private, max-age=300`
2. User A logs out, User B logs in on same browser
3. Browser serves User A's cached profile to User B (still within `max-age`)

**Fix:** Use `private, no-cache` + user-aware ETag:
```
Cache-Control: private, no-cache
ETag: "user42-profile-v3"
```

The ETag includes the user identifier, so `If-None-Match` will fail after account switch → server sends fresh data.

**Alternative:** `Vary: Cookie` forces a new cache entry when the session cookie changes. But this creates high-cardinality cache keys — prefer user-aware ETags.

## Cache Timing Attacks

An attacker can determine whether a user has previously visited a URL by measuring response time:
- Cached response: fast (~0ms network)
- Uncached response: slow (full network round-trip)

This reveals browsing history. Mitigations:
- Browsers are implementing cache partitioning (keyed by top-level site)
- Chrome, Firefox, and Safari all now partition HTTP cache by top-level origin
- `no-store` on sensitive resources prevents caching entirely

## HTTPS Requirement

Modern browsers restrict caching behavior over plain HTTP:
- Mixed content may not be cached
- Some browsers treat HTTP responses as implicitly `no-store`
- HSTS (Strict-Transport-Security) ensures HTTPS is always used

**Always serve cached content over HTTPS.** HTTP caching is unreliable and insecure.

## CDN Cache Poisoning via Missing `Vary`

If the response changes based on a request header, that header MUST be in `Vary`. Otherwise an attacker seeds the CDN cache with poisoned values served to all subsequent users.

**Real-world example (`X-Forwarded-Host` poisoning):**

1. App uses `X-Forwarded-Host` to generate asset URLs in HTML: `<script src="https://{host}/app.js">`
2. Attacker sends request with `X-Forwarded-Host: evil.com`
3. CDN caches the response: `<script src="https://evil.com/app.js">`
4. All subsequent users load and execute the attacker's script

**Fix:** Include the header in `Vary`, or better — don't trust `X-Forwarded-Host` at all. For CORS-specific `Vary: Origin` treatment, see [patterns.md](./patterns.md).

## Sensitive Data in Query Strings

URLs with sensitive data (tokens, API keys) may be cached by CDN if the response has caching headers:

```
# CDN may cache this with the token as part of the cache key
GET /api/data?token=secret123
Cache-Control: public, max-age=3600
```

CDN logs, analytics, and cache storage now contain the token. **Never put secrets in URLs.** Use `Authorization` header + `private` or `no-store`.
