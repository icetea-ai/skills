---
title: Transform Data with select Option
impact: HIGH
impactDescription: Enables partial subscriptions and automatic memoization
tags: data-transformation, select, performance, optimization
---

## Transform Data with select Option

Use the `select` option for data transformations to enable partial subscriptions and automatic memoization.

**Incorrect (transform in render):**

```typescript
const { data } = useQuery({
  queryKey: ['todos'],
  queryFn: fetchTodos
})

// ❌ Component re-renders whenever ANY field in data changes
const todoNames = data?.map(todo => todo.name.toUpperCase())
```

**Correct (transform with select):**

```typescript
const { data: todoNames } = useQuery({
  queryKey: ['todos'],
  queryFn: fetchTodos,
  select: (data) => data.map(todo => todo.name.toUpperCase())
})

// ✅ Component only re-renders when transformed output changes
```

**Partial subscriptions (powerful feature):**

```typescript
// Component A: Only cares about count
const { data: count } = useQuery({
  queryKey: ['todos'],
  queryFn: fetchTodos,
  select: (data) => data.length
})
// Re-renders only when count changes

// Component B: Only cares about names
const { data: names } = useQuery({
  queryKey: ['todos'],
  queryFn: fetchTodos,
  select: (data) => data.map(t => t.name)
})
// Re-renders only when names array changes

// Same query key = single network request, two selective subscriptions
```

**Memoize selector for stable reference:**

```typescript
// ❌ Inline function creates new reference every render
select: (data) => data.filter(t => t.completed)

// ✅ Extract to stable function
const selectCompleted = (data: Todo[]) => data.filter(t => t.completed)

useQuery({
  queryKey: ['todos'],
  queryFn: fetchTodos,
  select: selectCompleted
})

// ✅ Or useCallback for dynamic selectors
const selectByStatus = useCallback(
  (data: Todo[]) => data.filter(t => t.status === status),
  [status]
)
```

**Custom hook pattern:**

```typescript
function useTodos() {
  return useQuery({
    queryKey: ['todos'],
    queryFn: fetchTodos
  })
}

function useTodosCount() {
  return useQuery({
    queryKey: ['todos'],
    queryFn: fetchTodos,
    select: (data) => data.length
  })
}

// Different components can subscribe to different slices
```

**Why this matters:**

1. **Partial subscriptions** - Component re-renders only when its slice changes
2. **Automatic memoization** - React Query memoizes select output (structural sharing)
3. **Multiple observers** - Different components can transform same data differently
4. **Performance** - Reduces unnecessary re-renders

**Trade-offs:**

- `select` runs on every data change (structural sharing runs twice)
- Each observer can have different transformed structure
- DevTools show original data, not transformed

**When NOT to use select:**

```typescript
// ❌ Simple property access (no benefit)
select: (data) => data.user

// ❌ Already fast transformation
select: (data) => data.reverse()

// ✅ Use for meaningful transformations
select: (data) => data.map(item => ({
  ...item,
  computed: expensiveCalculation(item)
}))
```

**References:**
- [React Query Data Transformations](https://tkdodo.eu/blog/react-query-data-transformations)
- [React Query Render Optimizations](https://tkdodo.eu/blog/react-query-render-optimizations)
