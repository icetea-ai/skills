---
title: Prefer Invalidation Over Direct Updates
impact: HIGH
impactDescription: Safer and simpler cache management
tags: mutations, invalidation, cache-updates, setQueryData
---

## Prefer Invalidation Over Direct Updates

Use `invalidateQueries()` instead of `setQueryData()` after mutations - it's safer and avoids duplicating backend logic.

**Incorrect (manual cache updates):**

```typescript
const mutation = useMutation({
  mutationFn: updateTodo,
  onSuccess: (updatedTodo) => {
    // ❌ Manually updating cache
    queryClient.setQueryData(['todos', 'list'], (old) =>
      old?.map(todo => todo.id === updatedTodo.id ? updatedTodo : todo)
    )

    // ❌ What if list is sorted? New position might be wrong
    // ❌ What if there are multiple list variants (filtered)?
    // ❌ Duplicating backend sorting/filtering logic
  }
})
```

**Correct (invalidation with smart refetch):**

```typescript
const mutation = useMutation({
  mutationFn: updateTodo,
  onSuccess: () => {
    // ✅ Let backend determine correct state
    queryClient.invalidateQueries({ queryKey: ['todos', 'list'] })

    // Only active queries refetch automatically
    // Inactive queries marked stale (refetch on next mount)
  }
})
```

**Why invalidation is better:**

1. **No logic duplication** - Backend already knows sorting/filtering
2. **Handles edge cases** - Sorted lists, pagination, filters
3. **Always correct** - Single source of truth (backend)
4. **Simpler code** - No manual cache manipulation
5. **Smart refetching** - Only active queries refetch

**When to use setQueryData:**

```typescript
// ✅ Updating a single detail view (simple case)
const mutation = useMutation({
  mutationFn: updateTitle,
  onSuccess: (updatedTodo) => {
    // Safe: Just replacing one item
    queryClient.setQueryData(['todos', 'detail', updatedTodo.id], updatedTodo)

    // Still invalidate lists (position might change)
    queryClient.invalidateQueries({ queryKey: ['todos', 'list'] })
  }
})

// ✅ Optimistic updates (need immediate UI response)
const mutation = useMutation({
  mutationFn: toggleTodo,
  onMutate: async (id) => {
    await queryClient.cancelQueries({ queryKey: ['todos', id] })
    const previous = queryClient.getQueryData(['todos', id])

    queryClient.setQueryData(['todos', id], old => ({
      ...old,
      completed: !old.completed
    }))

    return { previous }
  },
  onError: (err, id, context) => {
    // Rollback on error
    queryClient.setQueryData(['todos', id], context.previous)
  }
})
```

**Hybrid approach (best of both):**

```typescript
const mutation = useMutation({
  mutationFn: updateTodo,
  onSuccess: (updatedTodo) => {
    // Update current detail view immediately
    queryClient.setQueryData(['todos', 'detail', updatedTodo.id], updatedTodo)

    // Invalidate other queries without refetching current view
    queryClient.invalidateQueries({
      queryKey: ['todos', 'list'],
      refetchType: 'none' // Mark stale but don't refetch
    })
  }
})
```

**Smart invalidation with fuzzy matching:**

```typescript
// Invalidate all todo-related queries
queryClient.invalidateQueries({ queryKey: ['todos'] })

// Matches:
// ['todos', 'list', { filter: 'all' }]
// ['todos', 'list', { filter: 'done' }]
// ['todos', 'detail', 5]
// All marked stale, active ones refetch
```

**References:**
- [Mastering Mutations in React Query](https://tkdodo.eu/blog/mastering-mutations-in-react-query)
- [Effective React Query Keys](https://tkdodo.eu/blog/effective-react-query-keys)
