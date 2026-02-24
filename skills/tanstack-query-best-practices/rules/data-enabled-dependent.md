---
title: Use enabled for Dependent Queries
impact: CRITICAL
impactDescription: Prevents queries running with invalid parameters
tags: dependent-queries, enabled, conditional-fetching
---

## Use enabled for Dependent Queries

When a query depends on data from another query, use the `enabled` option to prevent execution with invalid parameters.

**Incorrect (query runs with undefined):**

```typescript
const { data: user } = useQuery({
  queryKey: ['user'],
  queryFn: fetchUser
})

// ❌ Runs immediately with user?.id === undefined
const { data: projects } = useQuery({
  queryKey: ['projects', user?.id],
  queryFn: () => fetchProjects(user?.id) // Fails or fetches wrong data
})
```

**Correct (wait for dependency):**

```typescript
const { data: user } = useQuery({
  queryKey: ['user'],
  queryFn: fetchUser
})

// ✅ Only runs when user.id exists
const { data: projects } = useQuery({
  queryKey: ['projects', user?.id],
  queryFn: () => fetchProjects(user.id), // user.id guaranteed to exist
  enabled: !!user?.id
})
```

**With skipToken (v5.25+, type-safe):**

```typescript
import { skipToken } from '@tanstack/react-query'

const { data: user } = useQuery({
  queryKey: ['user'],
  queryFn: fetchUser
})

// ✅ TypeScript knows user.id is defined in queryFn
const { data: projects } = useQuery({
  queryKey: ['projects', user?.id],
  queryFn: user?.id
    ? () => fetchProjects(user.id)
    : skipToken
})
```

**Complex dependencies:**

```typescript
const enabled = !!(user?.id && user?.isActive && filters)

const { data } = useQuery({
  queryKey: ['data', user?.id, filters],
  queryFn: () => fetchData(user.id, filters),
  enabled
})
```

**Why this matters:**

Without `enabled`:
1. Query runs immediately with `undefined` parameters
2. API receives invalid requests
3. Error states or unexpected behavior
4. Wasted network requests

With `enabled`:
1. Query waits for valid dependencies
2. Automatically runs when enabled becomes `true`
3. No manual refetching needed
4. Type-safe with `skipToken`

**Pattern for multiple dependencies:**

```typescript
const { data: user } = useQuery(...)
const { data: settings } = useQuery(...)

const { data: dashboard } = useQuery({
  queryKey: ['dashboard', user?.id, settings?.theme],
  queryFn: () => fetchDashboard(user.id, settings.theme),
  enabled: !!(user?.id && settings?.theme)
})
```

**References:**
- [React Query and TypeScript](https://tkdodo.eu/blog/react-query-and-type-script)
- [Practical React Query](https://tkdodo.eu/blog/practical-react-query)
