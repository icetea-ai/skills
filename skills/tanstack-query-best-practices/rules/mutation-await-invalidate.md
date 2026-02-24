---
title: Return invalidateQueries Promise in onSuccess
impact: HIGH
impactDescription: Keeps mutation pending until refetch completes
tags: mutations, invalidation, loading-states, ux
---

## Return invalidateQueries Promise in onSuccess

Return the `invalidateQueries()` promise in `onSuccess` to keep mutation status as pending until refetch completes.

**Incorrect (fire-and-forget invalidation):**

```typescript
const mutation = useMutation({
  mutationFn: updateTodo,
  onSuccess: () => {
    // ❌ Not awaiting - mutation immediately shows success
    queryClient.invalidateQueries({ queryKey: ['todos'] })
  }
})

// UI shows success while refetch is still loading
// User sees stale data briefly before update appears
if (mutation.isPending) return <Spinner />
if (mutation.isSuccess) return <Success /> // ← Shows before data updates
```

**Correct (await invalidation):**

```typescript
const mutation = useMutation({
  mutationFn: updateTodo,
  onSuccess: () => {
    // ✅ Return promise - mutation stays pending until refetch done
    return queryClient.invalidateQueries({ queryKey: ['todos'] })
  }
})

// UI stays in pending state until fresh data is loaded
if (mutation.isPending) return <Spinner /> // ← Includes refetch time
if (mutation.isSuccess) return <Success /> // ← Data guaranteed fresh
```

**Visual timeline comparison:**

```typescript
// ❌ Without return:
mutate() → onSuccess() → isPending=false → [refetch happening]
                          ↑ Shows success here (data still stale)

// ✅ With return:
mutate() → onSuccess() → [refetch] → isPending=false
                                      ↑ Shows success here (data fresh)
```

**Multiple invalidations:**

```typescript
const mutation = useMutation({
  mutationFn: updateTodo,
  onSuccess: async () => {
    // ✅ Await all invalidations
    return Promise.all([
      queryClient.invalidateQueries({ queryKey: ['todos'] }),
      queryClient.invalidateQueries({ queryKey: ['stats'] })
    ])
  }
})
```

**Why this matters:**

Without `return`:
1. `isPending` becomes `false` immediately after mutation
2. Refetch happens in background
3. UI shows success with stale data
4. User sees "saved" but data hasn't updated yet
5. Confusing UX

With `return`:
1. `isPending` stays `true` until refetch completes
2. Loading spinner stays visible
3. Success state guarantees fresh data
4. Better UX - "saved" means data is actually updated

**Pattern for better UX:**

```typescript
const mutation = useMutation({
  mutationFn: updateTodo,
  onSuccess: () => {
    return queryClient.invalidateQueries({ queryKey: ['todos'] })
  }
})

function TodoForm() {
  const mutation = useUpdateTodo()

  return (
    <form onSubmit={(e) => {
      e.preventDefault()
      mutation.mutate(formData)
    }}>
      <button disabled={mutation.isPending}>
        {mutation.isPending ? 'Saving...' : 'Save'}
      </button>

      {/* Success shown only when data is fresh */}
      {mutation.isSuccess && <Toast>Saved successfully!</Toast>}
    </form>
  )
}
```

**When NOT to return:**

```typescript
// Background updates where immediate success is fine
onSuccess: () => {
  // Don't await - mark stale but don't block success
  queryClient.invalidateQueries({
    queryKey: ['analytics'],
    refetchType: 'none'
  })
}
```

**References:**
- [Mastering Mutations in React Query](https://tkdodo.eu/blog/mastering-mutations-in-react-query)
