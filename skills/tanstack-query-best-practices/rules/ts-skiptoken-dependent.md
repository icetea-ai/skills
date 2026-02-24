---
title: Use skipToken for Type-Safe Dependent Queries
impact: MEDIUM
impactDescription: Eliminates type errors in dependent queries
tags: typescript, skiptoken, dependent-queries, v5
---

## Use skipToken for Type-Safe Dependent Queries

Use `skipToken` (v5.25+) instead of `enabled` boolean for type-safe dependent queries.

**Problem with enabled:**

```typescript
const { data: user } = useQuery({
  queryKey: ['user'],
  queryFn: fetchUser
})

// ❌ Type error: user?.id might be undefined
const { data: projects } = useQuery({
  queryKey: ['projects', user?.id],
  queryFn: () => fetchProjects(user?.id), // Type error
  enabled: !!user?.id
})

// TypeScript doesn't know enabled prevents execution
```

**Solution with skipToken:**

```typescript
import { skipToken } from '@tanstack/react-query'

const { data: user } = useQuery({
  queryKey: ['user'],
  queryFn: fetchUser
})

// ✅ Type-safe: user.id guaranteed to exist
const { data: projects } = useQuery({
  queryKey: ['projects', user?.id],
  queryFn: user?.id
    ? () => fetchProjects(user.id)
    : skipToken
})
```

**How skipToken works:**

When `queryFn` is `skipToken`:
- Query is disabled (won't run)
- TypeScript knows the function branch won't execute
- No type errors about undefined parameters

**Multiple dependencies:**

```typescript
const { data: user } = useQuery(...)
const { data: settings } = useQuery(...)

// ✅ All dependencies checked
const { data } = useQuery({
  queryKey: ['dashboard', user?.id, settings?.theme],
  queryFn: user?.id && settings?.theme
    ? () => fetchDashboard(user.id, settings.theme)
    : skipToken
})
```

**Pattern comparison:**

```typescript
// ❌ enabled (type errors)
enabled: !!dependency,
queryFn: () => fetch(dependency) // Type error

// ⚠️ enabled + runtime check (verbose)
enabled: !!dependency,
queryFn: () => {
  if (!dependency) throw new Error('Invalid')
  return fetch(dependency)
}

// ⚠️ enabled + type assertion (unsafe)
enabled: !!dependency,
queryFn: () => fetch(dependency!)

// ✅ skipToken (type-safe and clean)
queryFn: dependency
  ? () => fetch(dependency)
  : skipToken
```

**When to use each:**

```typescript
// Use enabled: Complex boolean conditions
enabled: user?.isAdmin && hasPermission && !isLoading

// Use skipToken: Type-safe dependent queries
queryFn: userId ? () => fetchUser(userId) : skipToken

// Combine both:
enabled: user?.isAdmin, // Additional condition
queryFn: userId
  ? () => fetchUser(userId)
  : skipToken // Type safety
```

**Migration from enabled:**

```typescript
// Before (v5.24 and earlier)
const { data } = useQuery({
  queryKey: ['data', id],
  queryFn: () => fetchData(id!), // Type assertion
  enabled: !!id
})

// After (v5.25+)
const { data } = useQuery({
  queryKey: ['data', id],
  queryFn: id ? () => fetchData(id) : skipToken
})
```

**References:**
- [React Query and TypeScript](https://tkdodo.eu/blog/react-query-and-type-script)
- [TanStack Query skipToken docs](https://tanstack.com/query/latest/docs/react/guides/disabling-queries#typesafe-disabling-of-queries-using-skiptoken)
