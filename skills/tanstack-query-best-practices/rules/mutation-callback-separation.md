---
title: Separate Logic and UI Callbacks
impact: MEDIUM
impactDescription: Maintainable mutation side effects with proper separation of concerns
tags: mutations, callbacks, architecture, separation-of-concerns
---

## Separate Logic and UI Callbacks

Separate data-related logic (hook level) from UI-related side effects (component level) in mutation callbacks.

**Incorrect (mixing concerns):**

```typescript
// ❌ Everything in component
const mutation = useMutation({
  mutationFn: updateTodo,
  onSuccess: () => {
    queryClient.invalidateQueries({ queryKey: ['todos'] }) // Data
    history.push('/todos') // UI
    toast.success('Saved!') // UI
  }
})

// Can't reuse in components with different UI behavior
```

**Correct (separated concerns):**

```typescript
// ✅ Hook level: data logic (always runs)
const useUpdateTodo = () => {
  const queryClient = useQueryClient()

  return useMutation({
    mutationFn: updateTodo,
    onSuccess: () => {
      // Always invalidate - data integrity
      return queryClient.invalidateQueries({ queryKey: ['todos'] })
    }
  })
}

// ✅ Component level: UI effects (conditional)
function TodoForm() {
  const mutation = useUpdateTodo()

  const handleSubmit = (data) => {
    mutation.mutate(data, {
      onSuccess: () => {
        history.push('/todos')
        toast.success('Todo saved!')
      }
    })
  }
}

// Different component, different UI
function QuickEdit() {
  const mutation = useUpdateTodo()

  const handleSave = (data) => {
    mutation.mutate(data, {
      onSuccess: () => {
        closeModal()
        toast.success('Saved!')
      }
    })
  }
}
```

**Callback execution order:**

```typescript
// Hook callbacks fire BEFORE mutate callbacks
const mutation = useMutation({
  onSuccess: () => console.log('1. Hook level')
})

mutation.mutate(data, {
  onSuccess: () => console.log('2. Mutate level')
})

// Output: "1. Hook level" → "2. Mutate level"
```

**What goes where:**

**Hook level (useMutation config):**
- Cache invalidation/updates
- Analytics/logging
- Error reporting
- Always-needed side effects

**Component level (mutate options):**
- Navigation
- Toast notifications
- Modal/dialog control
- Form reset
- Component-specific state

**Component unmount consideration:**

```typescript
// Hook callbacks: Always run
const mutation = useMutation({
  onSuccess: () => {
    queryClient.invalidateQueries({ queryKey: ['todos'] })
    // ✅ Runs even if component unmounts
  }
})

// Mutate callbacks: May not run if unmounted
mutation.mutate(data, {
  onSuccess: () => {
    history.push('/todos')
    // ⚠️ Won't run if component unmounted
  }
})
```

**Pattern for global vs local:**

```typescript
const useCreateTodo = () => {
  const queryClient = useQueryClient()

  return useMutation({
    mutationFn: createTodo,

    // Global: Data integrity
    onSuccess: () => {
      return queryClient.invalidateQueries({ queryKey: ['todos'] })
    },

    // Global: Analytics
    onSettled: (data, error) => {
      analytics.track('todo_created', {
        success: !error,
        todoId: data?.id
      })
    }
  })
}

// Local: UI-specific
const mutation = useCreateTodo()

mutation.mutate(newTodo, {
  onSuccess: (data) => {
    if (shouldRedirect) {
      history.push(`/todos/${data.id}`)
    }
  },
  onError: (error) => {
    setFormError(error.message)
  }
})
```

**References:**
- [Mastering Mutations in React Query](https://tkdodo.eu/blog/mastering-mutations-in-react-query)
