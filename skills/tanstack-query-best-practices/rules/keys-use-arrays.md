---
title: Always Use Array Format for Query Keys
impact: CRITICAL
impactDescription: Prevents cache mismatches and enables hierarchical invalidation
tags: query-keys, architecture, caching
---

## Always Use Array Format for Query Keys

Query keys must be arrays to support hierarchical structure and proper cache matching.

**Incorrect (string keys deprecated in v4+):**

```typescript
useQuery({
  queryKey: 'todos',
  queryFn: fetchTodos
})
```

**Correct (array format):**

```typescript
useQuery({
  queryKey: ['todos'],
  queryFn: fetchTodos
})
```

**Why this matters:**

String keys get converted to arrays internally anyway in v4+. Using arrays from the start enables hierarchical structures:

```typescript
['todos'] // All todos
['todos', 'list'] // All lists
['todos', 'list', { filter: 'done' }] // Specific list
['todos', 'detail', 5] // Specific detail
```

This hierarchy allows granular cache control - invalidate all todos with `['todos']`, only lists with `['todos', 'list']`, or specific items.

**References:**
- [Effective React Query Keys](https://tkdodo.eu/blog/effective-react-query-keys)
