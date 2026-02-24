---
title: Avoid Hydration Mismatches with Streaming SSR
impact: HIGH
impactDescription: Prevents race conditions in v5.82.0+ streaming SSR
tags: ssr, hydration, streaming, nextjs, v5
---

## Avoid Hydration Mismatches with Streaming SSR

In v5.82.0+, streaming SSR can cause race conditions between `hydrate()` and `query.fetch()`. Use `await prefetchQuery` or `useSuspenseQuery`.

**The problem (v5.82.0+):**

```typescript
// Server Component (streaming)
export default async function Page() {
  const queryClient = getQueryClient()

  // ❌ Not awaited - streaming starts immediately
  queryClient.prefetchQuery({
    queryKey: ['data'],
    queryFn: getData
  })

  return <HydrationBoundary state={dehydrate(queryClient)}>
    <DataComponent />
  </HydrationBoundary>
}

// Error: "Hydration failed because the initial UI does not match"
```

**Why it happens:**

Race condition between server `prefetch` and client `hydrate()`:
- Server: prefetch starts → HTML streams → dehydrate()
- Client: hydrate() ← **race** → query.fetch()
- If query.fetch() wins, hydration mismatch occurs

**Solution 1: Await prefetch:**

```typescript
export default async function Page() {
  const queryClient = getQueryClient()

  // ✅ Await - ensures data in cache before streaming
  await queryClient.prefetchQuery({
    queryKey: ['data'],
    queryFn: getData
  })

  return <HydrationBoundary state={dehydrate(queryClient)}>
    <DataComponent />
  </HydrationBoundary>
}
```

**Solution 2: Use useSuspenseQuery:**

```typescript
function DataComponent() {
  // ✅ Coordinates with Suspense to prevent race
  const { data } = useSuspenseQuery({
    queryKey: ['data'],
    queryFn: getData
  })

  return <div>{data}</div>
}

export default async function Page() {
  return (
    <Suspense fallback={<Loading />}>
      <DataComponent />
    </Suspense>
  )
}
```

**Solution 3: Don't render based on fetchStatus:**

```typescript
function DataComponent() {
  // ❌ Causes hydration errors
  const { data, isFetching } = useSuspenseQuery({
    queryKey: ['data'],
    queryFn: getData
  })

  if (isFetching) return <Loading /> // ❌

  return <div>{data}</div>
}

// ✅ Let Suspense handle loading
function DataComponent() {
  const { data } = useSuspenseQuery({
    queryKey: ['data'],
    queryFn: getData
  })

  return <div>{data}</div>
}
```

**Complete Next.js App Router pattern:**

```typescript
// lib/get-query-client.ts
import { QueryClient } from '@tanstack/react-query'
import { cache } from 'react'

export const getQueryClient = cache(() => new QueryClient())

// app/data/page.tsx
export default async function DataPage() {
  const queryClient = getQueryClient()

  // ✅ Await all prefetches
  await Promise.all([
    queryClient.prefetchQuery({
      queryKey: ['user'],
      queryFn: fetchUser
    }),
    queryClient.prefetchQuery({
      queryKey: ['posts'],
      queryFn: fetchPosts
    })
  ])

  return (
    <HydrationBoundary state={dehydrate(queryClient)}>
      <Suspense fallback={<Skeleton />}>
        <DataDisplay />
      </Suspense>
    </HydrationBoundary>
  )
}
```

**Version check:**

Only affects v5.82.0+ with streaming SSR. If experiencing issues:
- Check version: `@tanstack/react-query: ^5.82.0`
- Apply one of the solutions above

**References:**
- [TanStack Query v5 Streaming SSR Issue](https://github.com/TanStack/query/issues/6606)
