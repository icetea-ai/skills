---
title: Structure Keys from Generic to Specific
impact: CRITICAL
impactDescription: Enables granular cache invalidation and updates
tags: query-keys, architecture, invalidation
---

## Structure Keys from Generic to Specific

Organize query keys in a hierarchy from generic to specific for flexible cache management.

**Incorrect (flat structure):**

```typescript
queryKey: ['todos-list-done']
queryKey: ['todos-detail-5']
```

**Correct (hierarchical structure):**

```typescript
// Lists
queryKey: ['todos', 'list', { filters: 'all' }]
queryKey: ['todos', 'list', { filters: 'done' }]

// Details
queryKey: ['todos', 'detail', 1]
queryKey: ['todos', 'detail', 2]
```

**Why this matters:**

Hierarchical keys enable granular cache control:

```typescript
// Invalidate ALL todo-related queries
queryClient.invalidateQueries({ queryKey: ['todos'] })

// Invalidate only list queries
queryClient.invalidateQueries({ queryKey: ['todos', 'list'] })

// Invalidate specific detail
queryClient.invalidateQueries({ queryKey: ['todos', 'detail', 5] })
```

React Query uses fuzzy matching - `['todos']` matches `['todos', 'list', { filters: 'done' }]` because it's a prefix.

**Pattern:**
1. Domain entity first (`'todos'`, `'users'`)
2. Query type second (`'list'`, `'detail'`, `'infinite'`)
3. Specifics last (filters, IDs, parameters)

**References:**
- [Effective React Query Keys](https://tkdodo.eu/blog/effective-react-query-keys)
