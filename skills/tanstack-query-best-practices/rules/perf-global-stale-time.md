---
title: Set Global staleTime Default
impact: HIGH
impactDescription: Dramatically reduces unnecessary network requests
tags: performance, configuration, staleTime, caching
---

## Set Global staleTime Default

Set a global `staleTime` (e.g. 20 seconds) to prevent excessive refetching on mount and window focus.

**Incorrect (default staleTime: 0):**

```typescript
const queryClient = new QueryClient()
// staleTime defaults to 0

// Every component mount triggers refetch
// Every window focus triggers refetch
// Excessive network requests for same data
```

**Correct (reasonable global default):**

```typescript
const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 20 * 1000, // 20 seconds
      gcTime: 5 * 60 * 1000, // 5 minutes
    },
  },
})
```

**Impact:**

With `staleTime: 0` (default):
```
Component A mounts → fetch
Component B mounts → fetch  (same data!)
Window focus → fetch
Navigate back → fetch
```

With `staleTime: 20000`:
```
Component A mounts → fetch
Component B mounts within 20s → use cache ✓
Window focus within 20s → use cache ✓
Navigate back within 20s → use cache ✓
```

**Override per-query when needed:**

```typescript
// Real-time data: refetch frequently
useQuery({
  queryKey: ['live-prices'],
  queryFn: fetchPrices,
  staleTime: 0, // Always refetch
  refetchInterval: 5000 // Poll every 5s
})

// Essentially static data
useQuery({
  queryKey: ['app-config'],
  queryFn: fetchConfig,
  staleTime: Infinity // Never refetch
})

// Most queries: use global default (20s)
useQuery({
  queryKey: ['todos'],
  queryFn: fetchTodos
  // staleTime: 20000 inherited
})
```

**Common patterns:**

```typescript
// User profile (rarely changes)
staleTime: 5 * 60 * 1000 // 5 minutes

// Dashboard data (moderate freshness)
staleTime: 30 * 1000 // 30 seconds

// Search results (balance freshness and UX)
staleTime: 10 * 1000 // 10 seconds

// Static reference data
staleTime: Infinity
```

**Why this matters:**

1. **Reduces network requests** - Orders of magnitude fewer fetches
2. **Faster UI** - Instant cache reads instead of loading spinners
3. **Better UX** - No flicker from unnecessary refetches
4. **Lower costs** - Fewer API calls
5. **Server relief** - Less load on backend

**Don't confuse with gcTime:**

```typescript
staleTime: 20000  // Controls REFETCH logic (when to fetch again)
gcTime: 300000    // Controls CACHE persistence (when to remove)

// Data lifecycle:
// 0-20s: Fresh (no refetch)
// 20s-5min: Stale (refetch on mount/focus) but cached
// 5min+: Garbage collected if inactive
```

**References:**
- [Practical React Query](https://tkdodo.eu/blog/practical-react-query)
- [TanStack Query Important Defaults](https://tanstack.com/query/latest/docs/react/guides/important-defaults)
