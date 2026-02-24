---
title: Leverage Automatic Property Tracking (v5)
impact: MEDIUM
impactDescription: Automatic re-render optimization without manual configuration
tags: performance, v5, tracked-queries, optimization
---

## Leverage Automatic Property Tracking (v5)

React Query v5 automatically tracks which properties you access during render and only re-renders when those properties change.

**How it works (v5 default):**

```typescript
const { data, isLoading } = useQuery({
  queryKey: ['todos'],
  queryFn: fetchTodos
})

// React Query tracks: data, isLoading
// Re-renders ONLY when these change

return (
  <div>
    {isLoading && <Spinner />}
    {data && <TodoList todos={data} />}
  </div>
)
```

**Background refetch example:**

```typescript
function TodoCount() {
  const { data } = useQuery({
    queryKey: ['todos'],
    queryFn: fetchTodos
  })

  // Only accessed 'data'
  // When background refetch starts:
  // - isFetching: false → true
  // - No re-render (didn't access isFetching)
  // When refetch completes:
  // - data changes → re-renders ✓

  return <div>Count: {data?.length}</div>
}
```

**Limitations:**

```typescript
// ❌ Object rest defeats tracking
const { isLoading, ...rest } = useQuery(...)
// Tracks ALL properties (rest captures everything)

// ✅ Destructure only what you need
const { data, isLoading, error } = useQuery(...)

// ❌ Accessing in useEffect without deps
useEffect(() => {
  console.log(query.data) // Not tracked properly
}, []) // Missing dependency

// ✅ Include in dependencies
useEffect(() => {
  console.log(query.data)
}, [query.data])
```

**Manual override when needed:**

```typescript
// Only care about data
useQuery({
  queryKey: ['todos'],
  queryFn: fetchTodos,
  notifyOnChangeProps: ['data']
})

// Background refetch won't trigger re-render
```

**Configuration:**

```typescript
// v5: Enabled by default
const queryClient = new QueryClient()

// Opt-out if needed
const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      notifyOnChangeProps: 'all', // Disable tracking
    },
  },
})
```

**Performance impact:**

```typescript
// Without tracking
const { data, isLoading, isFetching } = useQuery(...)
// Re-renders on ANY property change
// Including background refetch isFetching changes

// With tracking (v5 default)
const { data } = useQuery(...)
// Re-renders ONLY when data changes
// Background refetch doesn't cause re-render ✓
```

**Common pattern:**

```typescript
function TodoForm() {
  const mutation = useMutation({
    mutationFn: createTodo
  })

  // Only using isPending
  // Component doesn't re-render when error/data changes
  return (
    <form onSubmit={() => mutation.mutate(formData)}>
      <button disabled={mutation.isPending}>
        {mutation.isPending ? 'Saving...' : 'Save'}
      </button>
    </form>
  )
}
```

**References:**
- [React Query Render Optimizations](https://tkdodo.eu/blog/react-query-render-optimizations)
- [TanStack Query v5 Migration](https://tanstack.com/query/latest/docs/react/guides/migrating-to-v5)
