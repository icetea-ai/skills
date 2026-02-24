---
title: Memoize select Functions Properly
impact: MEDIUM
impactDescription: Prevents unnecessary selector re-runs
tags: performance, select, memoization, optimization
---

## Memoize select Functions Properly

Extract or memoize `select` functions to prevent them from running on every render.

**Incorrect (inline selector):**

```typescript
// ❌ New function created every render
// Selector runs every render even if data unchanged
useQuery({
  queryKey: ['todos'],
  queryFn: fetchTodos,
  select: (data) => data.filter(t => t.completed)
})
```

**Correct (stable function reference):**

```typescript
// ✅ Extract to module level
const selectCompleted = (data: Todo[]) => data.filter(t => t.completed)

useQuery({
  queryKey: ['todos'],
  queryFn: fetchTodos,
  select: selectCompleted
})

// Or with useCallback for dynamic selectors
const [status, setStatus] = useState('all')

const selectByStatus = useCallback(
  (data: Todo[]) => data.filter(t => t.status === status),
  [status] // Only recreate when status changes
)

useQuery({
  queryKey: ['todos'],
  queryFn: fetchTodos,
  select: selectByStatus
})
```

**Custom hook pattern:**

```typescript
// Reusable selector hooks
function useTodosQuery() {
  return useQuery({
    queryKey: ['todos'],
    queryFn: fetchTodos
  })
}

function useCompletedTodos() {
  return useQuery({
    queryKey: ['todos'],
    queryFn: fetchTodos,
    select: selectCompleted // Stable reference
  })
}

function useTodosByStatus(status: string) {
  return useQuery({
    queryKey: ['todos'],
    queryFn: fetchTodos,
    select: useCallback(
      (data) => data.filter(t => t.status === status),
      [status]
    )
  })
}
```

**When inline is acceptable:**

```typescript
// ✅ Simple property access (fast enough)
select: (data) => data.length

// ✅ Primitive transformation (negligible cost)
select: (data) => data.user.name

// ❌ Expensive computation (should memoize)
select: (data) => data.map(item => ({
  ...item,
  computed: expensiveCalculation(item)
}))
```

**How React Query memoizes select:**

```typescript
// React Query compares selector output using structural sharing
// Only re-renders if output changed (by reference)

const selectTodoIds = (data) => data.map(t => t.id)

// Even though selector runs on every data change,
// component only re-renders if output array is different
```

**Performance characteristics:**

```typescript
// Inline function (new reference every render)
select: (data) => transform(data)
// ⚠️ Selector runs every render
// ✅ Component only re-renders if output changed

// Stable function (same reference)
select: stableTransform
// ✅ Selector only runs when data changes
// ✅ Component only re-renders if output changed
```

**Pattern for complex selectors:**

```typescript
// Extract complex selectors to separate file
// selectors/todos.ts
export const selectCompleted = (todos: Todo[]) =>
  todos.filter(t => t.completed)

export const selectByPriority = (todos: Todo[]) =>
  todos.sort((a, b) => b.priority - a.priority)

export const selectStats = (todos: Todo[]) => ({
  total: todos.length,
  completed: todos.filter(t => t.completed).length,
  pending: todos.filter(t => !t.completed).length,
  highPriority: todos.filter(t => t.priority > 7).length
})

// Use in components
import { selectCompleted, selectStats } from './selectors/todos'

const { data: completed } = useTodos({ select: selectCompleted })
const { data: stats } = useTodos({ select: selectStats })
```

**References:**
- [React Query Data Transformations](https://tkdodo.eu/blog/react-query-data-transformations)
- [React Query Render Optimizations](https://tkdodo.eu/blog/react-query-render-optimizations)
