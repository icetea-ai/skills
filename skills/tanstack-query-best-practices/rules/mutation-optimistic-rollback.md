---
title: Implement Rollback for Optimistic Updates
impact: HIGH
impactDescription: Prevents UI showing incorrect state on mutation failure
tags: mutations, optimistic-updates, error-handling, rollback
---

## Implement Rollback for Optimistic Updates

Always implement rollback logic in `onError` when doing optimistic updates. Without it, UI shows incorrect state on failure.

**Incorrect (no rollback):**

```typescript
const mutation = useMutation({
  mutationFn: toggleTodo,
  onMutate: async (id) => {
    // Optimistically update UI
    queryClient.setQueryData(['todos', id], old => ({
      ...old,
      completed: !old.completed
    }))
    // ❌ If mutation fails, UI stuck in wrong state
  }
})
```

**Correct (with rollback):**

```typescript
const mutation = useMutation({
  mutationFn: toggleTodo,
  onMutate: async (id) => {
    // Cancel outgoing refetches
    await queryClient.cancelQueries({ queryKey: ['todos', id] })

    // Snapshot current state
    const previous = queryClient.getQueryData(['todos', id])

    // Optimistically update
    queryClient.setQueryData(['todos', id], old => ({
      ...old,
      completed: !old.completed
    }))

    // ✅ Return context for rollback
    return { previous, id }
  },
  onError: (err, id, context) => {
    // ✅ Rollback to previous state
    queryClient.setQueryData(['todos', context.id], context.previous)
    toast.error('Failed to update todo')
  },
  onSettled: (data, error, id) => {
    // ✅ Refetch to sync with backend
    queryClient.invalidateQueries({ queryKey: ['todos', id] })
  }
})
```

**Complete optimistic update pattern:**

```typescript
const useToggleTodo = () => {
  const queryClient = useQueryClient()

  return useMutation({
    mutationFn: toggleTodo,

    // 1. Before mutation starts
    onMutate: async (id) => {
      // Cancel any outgoing refetches
      await queryClient.cancelQueries({ queryKey: ['todos', id] })

      // Snapshot the previous value
      const previous = queryClient.getQueryData(['todos', id])

      // Optimistically update to new value
      queryClient.setQueryData(['todos', id], old => ({
        ...old,
        completed: !old.completed
      }))

      // Return context with previous value
      return { previous, id }
    },

    // 2. If mutation fails
    onError: (err, id, context) => {
      // Rollback to the previous value
      queryClient.setQueryData(['todos', context.id], context.previous)
    },

    // 3. Always refetch after error or success
    onSettled: (data, error, id) => {
      queryClient.invalidateQueries({ queryKey: ['todos', id] })
    }
  })
}
```

**Optimistic updates with lists:**

```typescript
const mutation = useMutation({
  mutationFn: deleteTodo,
  onMutate: async (idToDelete) => {
    await queryClient.cancelQueries({ queryKey: ['todos', 'list'] })

    const previous = queryClient.getQueryData(['todos', 'list'])

    // Optimistically remove from list
    queryClient.setQueryData(['todos', 'list'], old =>
      old?.filter(todo => todo.id !== idToDelete)
    )

    return { previous }
  },
  onError: (err, idToDelete, context) => {
    // Restore the list
    queryClient.setQueryData(['todos', 'list'], context.previous)
  },
  onSettled: () => {
    queryClient.invalidateQueries({ queryKey: ['todos', 'list'] })
  }
})
```

**Why this matters:**

Without rollback:
1. Mutation shows optimistic state
2. Server rejects mutation
3. UI still shows optimistic state
4. User thinks action succeeded but it didn't
5. Data inconsistency until manual refresh

With rollback:
1. Mutation shows optimistic state
2. Server rejects mutation
3. `onError` restores previous state
4. `onSettled` refetches to sync with server
5. UI always matches reality

**When NOT to use optimistic updates:**

```typescript
// ❌ Avoid for complex scenarios:
// - Sorted lists (position might be wrong)
// - Paginated data (which page does it go on?)
// - Data requiring server-generated IDs
// - Mutations with low success probability

// ✅ Good for optimistic updates:
// - Toggle buttons
// - Like/favorite actions
// - Simple status changes
// - High success probability mutations
```

**References:**
- [Mastering Mutations in React Query](https://tkdodo.eu/blog/mastering-mutations-in-react-query)
- [TanStack Query Docs: Optimistic Updates](https://tanstack.com/query/latest/docs/react/guides/optimistic-updates)
