---
title: Use mutate Over mutateAsync
impact: HIGH
impactDescription: Prevents unhandled promise rejections
tags: mutations, error-handling, promises, async
---

## Use mutate Over mutateAsync

Prefer `mutate()` over `mutateAsync()` - it handles errors automatically and is safer by default.

**Incorrect (unhandled rejection risk):**

```typescript
const mutation = useMutation({
  mutationFn: updateTodo
})

// ❌ If promise rejects and you don't catch, unhandled rejection
const handleSubmit = async () => {
  const data = await mutation.mutateAsync(newTodo)
  history.push(data.url) // Never runs if error occurs
}

// ❌ Easy to forget error handling
await mutation.mutateAsync(newTodo) // Crashes app on error
```

**Correct (safe error handling):**

```typescript
const mutation = useMutation({
  mutationFn: updateTodo
})

// ✅ Errors handled automatically (noop internally)
const handleSubmit = () => {
  mutation.mutate(newTodo, {
    onSuccess: (data) => {
      history.push(data.url)
    },
    onError: (error) => {
      toast.error(error.message)
    }
  })
}
```

**Behind the scenes:**

```typescript
// React Query essentially does:
mutate = (variables, callbacks) => {
  mutateAsync(variables)
    .then(callbacks.onSuccess)
    .catch(callbacks.onError)
    .catch(noop) // ← This prevents unhandled rejections
}
```

**When to use mutateAsync:**

```typescript
// ✅ Dependent mutations requiring Promise composition
const handleComplex = async () => {
  try {
    const user = await createUser.mutateAsync(userData)
    const profile = await createProfile.mutateAsync({
      userId: user.id,
      ...profileData
    })
    history.push(`/users/${profile.id}`)
  } catch (error) {
    toast.error(error.message)
  }
}

// ✅ Concurrent mutations
const handleBatch = async () => {
  try {
    const results = await Promise.all([
      mutation1.mutateAsync(data1),
      mutation2.mutateAsync(data2),
      mutation3.mutateAsync(data3)
    ])
    toast.success('All saved!')
  } catch (error) {
    toast.error('Save failed')
  }
}
```

**Why this matters:**

`mutate()`:
- ✅ Safe by default
- ✅ No unhandled rejections possible
- ✅ Callbacks for side effects
- ✅ Recommended for 90% of cases

`mutateAsync()`:
- ⚠️ Requires manual error handling
- ⚠️ Easy to forget `.catch()`
- ⚠️ Causes unhandled rejections if errors aren't caught
- ✅ Needed for Promise composition

**Pattern for both worlds:**

```typescript
// Hook level: data-related logic
const useUpdateTodo = () => {
  const queryClient = useQueryClient()

  return useMutation({
    mutationFn: updateTodo,
    onSuccess: () => {
      // Always invalidate (data logic)
      queryClient.invalidateQueries({ queryKey: ['todos'] })
    }
  })
}

// Component level: UI-related side effects
const mutation = useUpdateTodo()

mutation.mutate(data, {
  onSuccess: () => {
    // Conditional navigation (UI logic)
    history.push('/todos')
  }
})
```

**References:**
- [Mastering Mutations in React Query](https://tkdodo.eu/blog/mastering-mutations-in-react-query)
