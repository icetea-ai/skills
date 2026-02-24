---
title: Derive UI State During Render, Not Effects
impact: CRITICAL
impactDescription: Prevents unnecessary re-renders and stale derived state
tags: data-flow, derived-state, performance, react-patterns
---

## Derive UI State During Render, Not Effects

Calculate derived state during render, not in `useEffect` with `useState`. Effects run after render, causing extra re-renders.

**Incorrect (derived state in effect):**

```typescript
const { data: todos } = useQuery({
  queryKey: ['todos'],
  queryFn: fetchTodos
})

const [completedCount, setCompletedCount] = useState(0)

useEffect(() => {
  if (todos) {
    setCompletedCount(todos.filter(t => t.completed).length) // ❌ Extra render
  }
}, [todos])

// First render: completedCount is stale
// Second render: completedCount is updated
```

**Correct (derive during render):**

```typescript
const { data: todos } = useQuery({
  queryKey: ['todos'],
  queryFn: fetchTodos
})

// ✅ Computed during render - always correct
const completedCount = todos?.filter(t => t.completed).length ?? 0

// Single render with correct value
```

**With expensive computations:**

```typescript
const { data: todos } = useQuery({
  queryKey: ['todos'],
  queryFn: fetchTodos
})

// ✅ Memoize expensive derivations
const completedCount = useMemo(
  () => todos?.filter(t => t.completed).length ?? 0,
  [todos]
)
```

**Why this matters:**

1. **One render instead of two** - No effect → setState → re-render cycle
2. **Always in sync** - Derived value can't be stale
3. **Simpler code** - No effect dependencies to maintain
4. **React 18 concurrent safety** - Derived values guaranteed consistent within render

**Pattern for complex derivations:**

```typescript
// ✅ Extract to custom hook if complex
function useTodoStats(todos) {
  return useMemo(() => ({
    total: todos?.length ?? 0,
    completed: todos?.filter(t => t.completed).length ?? 0,
    pending: todos?.filter(t => !t.completed).length ?? 0,
  }), [todos])
}

const { data: todos } = useTodos()
const stats = useTodoStats(todos)
```

**References:**
- [React Query as a State Manager](https://tkdodo.eu/blog/react-query-as-a-state-manager)
- [React Docs: You Might Not Need an Effect](https://react.dev/learn/you-might-not-need-an-effect#updating-state-based-on-props-or-state)
