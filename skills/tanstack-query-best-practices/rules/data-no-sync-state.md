---
title: Never Sync Server State to useState
impact: CRITICAL
impactDescription: Prevents dual source of truth and synchronization bugs
tags: data-flow, state-management, anti-pattern, useState
---

## Never Sync Server State to useState

Server state from `useQuery` should never be copied to `useState` via `useEffect`. This creates dual source of truth.

**Incorrect (dual source of truth):**

```typescript
const { data } = useQuery({
  queryKey: ['todos'],
  queryFn: fetchTodos
})

const [todos, setTodos] = useState([])

useEffect(() => {
  if (data) {
    setTodos(data) // ❌ Now have two sources of truth
  }
}, [data])

// Which is correct? data or todos?
```

**Correct (single source of truth):**

```typescript
const { data } = useQuery({
  queryKey: ['todos'],
  queryFn: fetchTodos
})

// Use data directly
return <TodoList todos={data} />
```

**For form initialization:**

```typescript
// ❌ Don't sync to useState
useEffect(() => {
  if (data) setFormState(data)
}, [data])

// ✅ Use defaultValues or key the form
<Form key={data.id} defaultValues={data} />

// Or with react-hook-form
const { reset } = useForm()
useEffect(() => {
  if (data) reset(data) // Reset form, not manual state sync
}, [data, reset])
```

**Why this matters:**

1. **Single source of truth** - `data` is always correct
2. **Automatic updates** - Background refetches update UI
3. **No sync bugs** - Can't have `data` and `todos` out of sync
4. **Cache benefits** - Structural sharing, stale-while-revalidate work correctly

**When you think you need local state:**
- Form initialization → Use `defaultValues` or `reset()`
- Derived UI state → Compute during render
- Optimistic updates → Use `useMutation` with `onMutate`

**References:**
- [Practical React Query](https://tkdodo.eu/blog/practical-react-query)
- [React Query as a State Manager](https://tkdodo.eu/blog/react-query-as-a-state-manager)
