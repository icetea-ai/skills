---
title: Don't Render Based on isFetching with Suspense
impact: HIGH
impactDescription: Prevents hydration mismatches in Suspense boundaries
tags: ssr, suspense, hydration, loading-states
---

## Don't Render Based on isFetching with Suspense

Never conditionally render based on `isFetching` when using Suspense boundaries - it causes hydration mismatches.

**The problem:**

```typescript
function DataComponent() {
  const { data, isFetching } = useSuspenseQuery({
    queryKey: ['data'],
    queryFn: getData
  })

  // ❌ Causes hydration errors
  if (isFetching) return <Spinner />

  return <div>{data}</div>
}

// Error: "Hydration failed because the initial UI does not match"
```

**Why it fails:**

Server renders `<div>data</div>` (isFetching: false), but client might render `<Spinner />` (isFetching: true during hydration), causing a mismatch.

**Solution: Let Suspense handle loading:**

```typescript
function DataComponent() {
  // ✅ Only destructure data
  const { data } = useSuspenseQuery({
    queryKey: ['data'],
    queryFn: getData
  })

  // ✅ Always render same structure
  return <div>{data}</div>
}

// Suspense boundary handles loading
<Suspense fallback={<Spinner />}>
  <DataComponent />
</Suspense>
```

**For background refetch indicators:**

```typescript
function DataComponent() {
  const { data, isFetching } = useSuspenseQuery({
    queryKey: ['data'],
    queryFn: getData
  })

  return (
    <div>
      {/* ✅ Always render main content */}
      <Content data={data} />

      {/* ✅ Additive indicator (doesn't change structure) */}
      {isFetching && (
        <div className="fixed top-4 right-4">
          <RefreshIcon className="animate-spin" />
        </div>
      )}
    </div>
  )
}
```

**Anti-patterns:**

```typescript
// ❌ Conditional rendering
if (isFetching) return <Loading />

// ❌ Conditional content
{isFetching ? <Spinner /> : <Content />}

// ❌ Changing layout
<div className={isFetching ? 'loading' : 'loaded'}>
  {isFetching ? <Skeleton /> : <Content />}
</div>
```

**Patterns that work:**

```typescript
// ✅ Additive indicators
<div>
  <Content />
  {isFetching && <OverlaySpinner />}
</div>

// ✅ Style changes only
<div className={isFetching ? 'opacity-50' : ''}>
  <Content />
</div>

// ✅ Separate indicator element
<div>
  <Content />
</div>
{isFetching && <TopBarIndicator />}
```

**Suspense handles states automatically:**

```typescript
// Initial loading: Suspense shows fallback
<Suspense fallback={<Skeleton />}>
  <DataComponent />
</Suspense>

// Background refetch: Component stays mounted
// Use isFetching for optional indicator only
```

**References:**
- [TanStack Query Suspense Guide](https://tanstack.com/query/latest/docs/react/guides/suspense)
