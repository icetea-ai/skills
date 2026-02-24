---
title: Use Centralized Query Key Factory
impact: CRITICAL
impactDescription: Type-safe key management with DRY hierarchy
tags: query-keys, architecture, typescript, maintainability
---

## Use Centralized Query Key Factory

Create a centralized factory object for managing query keys with type safety and hierarchical relationships.

**Incorrect (hardcoded strings scattered):**

```typescript
// In component A
useQuery({ queryKey: ['todos', 'list', filter] })

// In component B
useQuery({ queryKey: ['todos', 'detail', id] })

// In mutation
queryClient.invalidateQueries({ queryKey: ['todos', 'list'] }) // Typo risk
```

**Correct (centralized factory):**

```typescript
const todoKeys = {
  all: ['todos'] as const,
  lists: () => [...todoKeys.all, 'list'] as const,
  list: (filters: string) => [...todoKeys.lists(), { filters }] as const,
  details: () => [...todoKeys.all, 'detail'] as const,
  detail: (id: number) => [...todoKeys.details(), id] as const,
}

// Usage in components
useQuery({
  queryKey: todoKeys.list(filter),
  queryFn: () => fetchTodos(filter)
})

useQuery({
  queryKey: todoKeys.detail(id),
  queryFn: () => fetchTodo(id)
})

// Usage in mutations
queryClient.invalidateQueries({ queryKey: todoKeys.lists() })
```

**Why this matters:**

1. **Type safety** - TypeScript ensures correct parameters
2. **DRY hierarchy** - Changes propagate automatically
3. **Refactoring safety** - Rename in one place
4. **Autocomplete** - IDE suggests available keys
5. **No typos** - Compile-time errors instead of runtime bugs

**Granular invalidation with factory:**

```typescript
// Remove all todo data
queryClient.removeQueries({ queryKey: todoKeys.all })

// Invalidate all lists (any filters)
queryClient.invalidateQueries({ queryKey: todoKeys.lists() })

// Prefetch specific detail
queryClient.prefetchQuery({
  queryKey: todoKeys.detail(id),
  queryFn: () => fetchTodo(id)
})
```

**References:**
- [Effective React Query Keys](https://tkdodo.eu/blog/effective-react-query-keys)
