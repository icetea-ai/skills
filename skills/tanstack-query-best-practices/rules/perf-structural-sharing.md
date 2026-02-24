---
title: Understand Structural Sharing Behavior
impact: MEDIUM
impactDescription: Know when React Query preserves object references
tags: performance, structural-sharing, references, optimization
---

## Understand Structural Sharing Behavior

React Query uses structural sharing to preserve referential equality of unchanged data. Understand when it works and when to disable it.

**What is structural sharing:**

```typescript
const oldData = { items: [{ id: 1, name: 'A' }, { id: 2, name: 'B' }] }
const newData = { items: [{ id: 1, name: 'A' }, { id: 2, name: 'C' }] }

// Without structural sharing
oldData.items[0] !== newData.items[0] // New reference (even though unchanged!)

// With structural sharing (React Query default)
oldData.items[0] === newData.items[0] // ✅ Same reference (unchanged)
oldData.items[1] !== newData.items[1] // New reference (changed)
```

**Why it matters:**

```typescript
function ItemDetail({ id }: { id: 1 }) {
  const { data } = useQuery({
    queryKey: ['items'],
    queryFn: fetchItems
  })

  const item = data?.items[0]

  // With structural sharing:
  // - Background refetch happens
  // - item[0] keeps same reference (if unchanged)
  // - Component doesn't re-render ✓

  return <div>{item?.name}</div>
}
```

**How it works:**

Recursively compares old and new data:
- Same value → keep old reference
- Different value → use new reference
- Works on plain objects, arrays, primitives
- Doesn't work on class instances, Maps, Sets, functions

**When to disable:**

```typescript
// Large datasets (structural sharing is expensive)
useQuery({
  queryKey: ['huge-dataset'],
  queryFn: fetchHugeDataset,
  structuralSharing: false
})

// Non-JSON data
useQuery({
  queryKey: ['class-instances'],
  queryFn: fetchClassInstances,
  structuralSharing: false // Class instances, Maps, Sets
})
```

**With select:**

```typescript
const { data } = useQuery({
  queryKey: ['todos'],
  queryFn: fetchTodos,
  select: (data) => data.map(t => ({ id: t.id, name: t.name }))
})

// Structural sharing runs TWICE:
// 1. fetchTodos result vs cached data
// 2. select output vs previous select output
```

**Performance trade-offs:**

```typescript
// Small datasets: Cheap, high benefit
const { data } = useQuery({
  queryKey: ['user'],
  queryFn: fetchUser
  // structuralSharing: true (default) ✅
})

// Large datasets: Expensive, may not be worth it
const { data } = useQuery({
  queryKey: ['10k-items'],
  queryFn: fetch10kItems,
  structuralSharing: false // ✅ Skip overhead
})
```

**Limitations:**

```typescript
// ❌ Doesn't work with:
// - Class instances
// - Functions
// - Symbols
// - Maps/Sets
// - Circular references

// ✅ Works with:
// - Plain objects
// - Arrays
// - Primitives (string, number, boolean, null)
```

**References:**
- [React Query Render Optimizations](https://tkdodo.eu/blog/react-query-render-optimizations)
- [TanStack Query Structural Sharing](https://tanstack.com/query/latest/docs/react/guides/render-optimizations#structural-sharing)
