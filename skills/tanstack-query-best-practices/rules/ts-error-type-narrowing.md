---
title: Handle Error Type Narrowing Properly
impact: MEDIUM
impactDescription: Type-safe error handling without type assertions
tags: typescript, error-handling, type-narrowing
---

## Handle Error Type Narrowing Properly

Use runtime type checking or module augmentation for type-safe error handling.

**The problem:**

```typescript
const { error } = useQuery({
  queryKey: ['todos'],
  queryFn: fetchTodos
})

// error type is Error | null (v5) or unknown (v4)
if (error) {
  console.log(error.response) // ❌ Type error
}
```

**Solution 1: Runtime type checking:**

```typescript
import { AxiosError } from 'axios'

const { error } = useQuery({
  queryKey: ['todos'],
  queryFn: fetchTodos
})

// ✅ Type guard narrows error type
if (error instanceof AxiosError) {
  console.log(error.response?.status)
  console.log(error.response?.data)
}
```

**Solution 2: Module augmentation (global):**

```typescript
// types/react-query.d.ts
declare module '@tanstack/react-query' {
  interface Register {
    defaultError: AxiosError
  }
}

// Now all queries use AxiosError
const { error } = useQuery({
  queryKey: ['todos'],
  queryFn: fetchTodos
})

// error is AxiosError | null (no runtime check needed)
if (error) {
  console.log(error.response?.status) // ✅ Type-safe
}
```

**Custom error types:**

```typescript
class ApiError extends Error {
  constructor(
    public status: number,
    public code: string,
    message: string
  ) {
    super(message)
  }
}

function isApiError(error: unknown): error is ApiError {
  return error instanceof ApiError
}

// Usage
const { error } = useQuery({
  queryKey: ['data'],
  queryFn: async () => {
    const res = await fetch('/api/data')
    if (!res.ok) {
      throw new ApiError(res.status, 'FETCH_ERROR', 'Failed')
    }
    return res.json()
  }
})

if (isApiError(error)) {
  if (error.status === 404) return <NotFound />
  if (error.status >= 500) return <ServerError code={error.code} />
}
```

**Version differences:**

```typescript
// v4: Error type defaults to unknown
const { error } = useQuery(...) // error: unknown

// v5: Error type defaults to Error
const { error } = useQuery(...) // error: Error | null

// Both: Override with module augmentation
declare module '@tanstack/react-query' {
  interface Register {
    defaultError: AxiosError
  }
}
```

**References:**
- [React Query and TypeScript](https://tkdodo.eu/blog/react-query-and-type-script)
- [React Query Error Handling](https://tkdodo.eu/blog/react-query-error-handling)
