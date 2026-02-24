---
title: Understand staleTime vs gcTime
impact: CRITICAL
impactDescription: Controls refetch behavior and cache persistence correctly
tags: configuration, caching, performance, staleTime, gcTime
---

## Understand staleTime vs gcTime

`staleTime` and `gcTime` control different aspects of caching. Confusing them causes over-fetching or data loss.

**staleTime (default: 0):**
- How long data is considered **fresh**
- Fresh data = no background refetch on mount/focus
- Controls **refetch logic**

**gcTime (default: 5 minutes):**
- How long **unused** data stays in cache
- After gcTime, cache entry is garbage collected
- Controls **cache persistence**

**Incorrect (confusing the two):**

```typescript
// ❌ Thinking gcTime prevents refetching
useQuery({
  queryKey: ['todos'],
  queryFn: fetchTodos,
  gcTime: 60000 // Won't prevent refetches!
})
```

**Correct (using both appropriately):**

```typescript
// ✅ Set staleTime to prevent refetching
const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 20000, // Data fresh for 20s
      gcTime: 5 * 60 * 1000, // Keep in cache for 5 min
    },
  },
})

// Data lifecycle:
// 0-20s: Fresh (no background refetch)
// 20s-5min: Stale (refetch on mount/focus) but in cache
// 5min+: Garbage collected if inactive
```

**Common patterns:**

```typescript
// Essentially static data
staleTime: Infinity
gcTime: Infinity

// Moderately fresh data (recommended default)
staleTime: 20000 // 20 seconds
gcTime: 5 * 60 * 1000 // 5 minutes

// Always fresh (real-time data)
staleTime: 0 // Default
refetchInterval: 5000 // Poll every 5s
```

**Why this matters:**

With `staleTime: 0` (default):
- Every mount triggers background refetch
- Every window focus triggers background refetch
- Causes excessive network requests

With `staleTime: 20000`:
- Mounting within 20s uses cache (no fetch)
- Focus within 20s uses cache (no fetch)
- Dramatically reduces requests

**Visual timeline:**

```
Fetch → [Fresh: 0-20s] → [Stale: 20s-5min] → [GC: 5min+]
         ↑ No refetch      ↑ Refetch on        ↑ Removed
                             mount/focus
```

**References:**
- [Practical React Query](https://tkdodo.eu/blog/practical-react-query)
- [TanStack Query Docs: Important Defaults](https://tanstack.com/query/latest/docs/react/guides/important-defaults)
