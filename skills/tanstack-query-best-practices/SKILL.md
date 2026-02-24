---
name: tanstack-query-best-practices
description: |
  Use when implementing, reviewing, or debugging TanStack Query features—including hydration/SSR flows and MSW-backed testing—to resolve cache drift, hydration mismatches, or flaky hooks across useQuery/useMutation and QueryClient orchestration.
---

# TanStack Query Best Practices

Comprehensive best practices for TanStack Query (React Query), distilled from TkDodo's "Practical React Query" series. Contains 27 essential rules across 7 categories, prioritized by impact.

## Overview

This guide packages TkDodo's field-tested guardrails so TanStack Query caches stay trustworthy, SSR hydration stays in sync, and automated tests stop flaking. Use it to diagnose cache drift, stale data churn, or hydration mismatch warnings, and to align team code reviews on a consistent rule set.

## When to Apply

Look here when any of these symptoms show up:
- Cache states diverge between tabs/devices or stale renders keep flashing old data
- SSR routes throw hydration mismatch warnings or Suspense waterfalls during streaming
- Mutations cause optimistic update drift or invalidateQueries never settles
- Tests flake because retries hide failures or QueryClient instances leak across suites
- Team code reviews debate query key shapes, staleTime vs. gcTime, or when to derive client state

**When not to use:** If you're building plain REST hooks without TanStack Query or treating all server data as static config, lighter guidance is sufficient.

## Rule Categories by Priority

| Priority | Category | Impact | Prefix | Rule Count |
|----------|----------|--------|--------|------------|
| 1 | Query Keys & Architecture | CRITICAL | `keys-` | 4 rules |
| 2 | Data Flow & Integrity | CRITICAL | `data-` | 5 rules |
| 3 | Mutations & Updates | HIGH | `mutation-` | 5 rules |
| 4 | Performance & Optimization | HIGH | `perf-` | 4 rules |
| 5 | TypeScript & Type Safety | MEDIUM | `ts-` | 3 rules |
| 6 | SSR & Hydration | HIGH | `ssr-` | 3 rules |
| 7 | Testing & Reliability | CRITICAL | `test-` | 3 rules |

## Quick Reference

### 1. Query Keys & Architecture (CRITICAL)

- `keys-use-arrays` — Always use array format for query keys
- `keys-hierarchical` — Structure keys from generic to specific
- `keys-factory-pattern` — Use centralized Query Key Factory
- `keys-as-dependencies` — Treat keys like dependency arrays

**Key insight:** Query keys are the foundation—everything else breaks without proper key management.

### 2. Data Flow & Integrity (CRITICAL)

- `data-no-sync-state` — Never sync server state to useState
- `data-derive-during-render` — Derive UI state during render, not effects
- `data-stale-vs-gc` — Understand staleTime vs gcTime
- `data-enabled-dependent` — Use enabled for dependent queries
- `data-select-transform` — Transform data with select option

**Key insight:** Server state should flow declaratively, not be managed imperatively.

### 3. Mutations & Updates (HIGH)

- `mutation-prefer-mutate` — Use mutate over mutateAsync
- `mutation-invalidate-vs-update` — Prefer invalidation over direct updates
- `mutation-await-invalidate` — Return invalidateQueries promise
- `mutation-optimistic-rollback` — Implement rollback for optimistic updates
- `mutation-callback-separation` — Separate logic and UI callbacks

**Key insight:** Invalidation is safer than manual cache updates—let the backend determine correct state.

### 4. Performance & Optimization (HIGH)

- `perf-global-stale-time` — Set global staleTime default (e.g., 20s)
- `perf-select-memoization` — Memoize select functions properly
- `perf-structural-sharing` — Understand structural sharing behavior
- `perf-tracked-queries` — Leverage automatic property tracking (v5)

**Key insight:** `staleTime: 0` (default) causes excessive refetching—set a global default.

### 5. TypeScript & Type Safety (MEDIUM)

- `ts-infer-from-queryfn` — Let TypeScript infer types from queryFn
- `ts-skiptoken-dependent` — Use skipToken for type-safe dependent queries
- `ts-error-type-narrowing` — Handle error type narrowing properly

**Key insight:** Type the queryFn, not the hook—inference flows automatically.

### 6. SSR & Hydration (HIGH)

- `ssr-suspense-query` — Use useSuspenseQuery for SSR routes
- `ssr-streaming-hydration` — Avoid hydration mismatches with streaming SSR
- `ssr-no-fetchstatus-suspense` — Don't render based on isFetching with Suspense

**Key insight:** Use `useSuspenseQuery` with SSR prefetching to prevent hydration errors.

### 7. Testing & Reliability (CRITICAL)

- `test-unique-client` — Create new QueryClient per test
- `test-disable-retries` — Disable retries in test environment
- `test-msw-mocking` — Use Mock Service Worker for network mocking

**Key insight:** Shared QueryClient = flaky tests. Retries enabled = test timeouts.

## How to Use

- Need to decode an error string? Jump to [ERROR-INDEX.md](./ERROR-INDEX.md) for the right rule.
- Looking for specific guardrails? Browse the [`rules/`](./rules/) directory (e.g., `rules/keys-use-arrays.md`) for deep dives with code samples.
- Need the full narrative? Review this SKILL plus individual [`rules/`](./rules/) files—this repository no longer maintains a separate AGENTS.md bundle.

## Common Patterns

### Query Key Factory

```typescript
const todoKeys = {
  all: ['todos'] as const,
  lists: () => [...todoKeys.all, 'list'] as const,
  list: (filters: string) => [...todoKeys.lists(), { filters }] as const,
  details: () => [...todoKeys.all, 'detail'] as const,
  detail: (id: number) => [...todoKeys.details(), id] as const,
}
```

### Custom Hook with Type Inference

```typescript
function useTodos(filters?: string) {
  return useQuery({
    queryKey: todoKeys.list(filters),
    queryFn: () => fetchTodos(filters),
    select: (data) => data.filter(t => !t.deleted)
  })
}
```

### Safe Mutation Pattern

```typescript
const mutation = useMutation({
  mutationFn: updateTodo,
  onSuccess: () => {
    return queryClient.invalidateQueries({ queryKey: todoKeys.lists() })
  }
})

mutation.mutate(data, {
  onSuccess: () => history.push('/todos')
})
```

### SSR with Next.js App Router

```typescript
// Server Component
export default async function Page() {
  const queryClient = getQueryClient()

  await queryClient.prefetchQuery({
    queryKey: ['data'],
    queryFn: getData
  })

  return <HydrationBoundary state={dehydrate(queryClient)}>
    <Suspense fallback={<Loading />}>
      <DataComponent />
    </Suspense>
  </HydrationBoundary>
}

// Client Component
function DataComponent() {
  const { data } = useSuspenseQuery({
    queryKey: ['data'],
    queryFn: getData
  })

  return <div>{data}</div>
}
```

### Test Setup

```typescript
const createWrapper = () => {
  const queryClient = new QueryClient({
    defaultOptions: {
      queries: {
        retry: false,
        staleTime: 20000,
      },
    },
  })

  return ({ children }) => (
    <QueryClientProvider client={queryClient}>
      {children}
    </QueryClientProvider>
  )
}
```

## References

- Author: tkdodo-distilled · Version 1.0.0 · Last updated 2026-02-03
- Source inspiration: TkDodo's "Practical React Query" series

TkDodo highlights:
- [Practical React Query](https://tkdodo.eu/blog/practical-react-query)
- [Effective React Query Keys](https://tkdodo.eu/blog/effective-react-query-keys)
- [React Query Data Transformations](https://tkdodo.eu/blog/react-query-data-transformations)
- [Mastering Mutations in React Query](https://tkdodo.eu/blog/mastering-mutations-in-react-query)
- [React Query and TypeScript](https://tkdodo.eu/blog/react-query-and-type-script)
- [Testing React Query](https://tkdodo.eu/blog/testing-react-query)

Official documentation:
- [TanStack Query Documentation](https://tanstack.com/query/latest/docs/react/overview)
