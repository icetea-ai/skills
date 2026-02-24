---
title: Use useSuspenseQuery for SSR Routes
impact: HIGH
impactDescription: Prevents hydration mismatches with server-prefetched data
tags: ssr, hydration, suspense, nextjs
---

## Use useSuspenseQuery for SSR Routes

Use `useSuspenseQuery` instead of `useQuery` when server-prefetching data to prevent hydration mismatches.

**The problem:**

```typescript
// Server Component
export default async function Page() {
  const queryClient = getQueryClient()

  await queryClient.prefetchQuery({
    queryKey: ['todos'],
    queryFn: fetchTodos
  })

  return <HydrationBoundary state={dehydrate(queryClient)}>
    <Todos />
  </HydrationBoundary>
}

// Client Component
function Todos() {
  // ❌ Hydration mismatch!
  const { data } = useQuery({
    queryKey: ['todos'],
    queryFn: fetchTodos
  })

  return <div>{data?.length} todos</div>
}

// Error: "Text content does not match server-rendered HTML"
```

**Why it happens:**

Server renders `<div>5 todos</div>`, but client might initially render `<div> todos</div>` (data undefined) before hydrating, causing a mismatch.

**Solution: useSuspenseQuery:**

```typescript
function Todos() {
  // ✅ Hydration-safe
  const { data } = useSuspenseQuery({
    queryKey: ['todos'],
    queryFn: fetchTodos
  })

  // data is never undefined
  return <div>{data.length} todos</div>
}
```

**Complete Next.js App Router pattern:**

```typescript
// app/todos/page.tsx (Server Component)
import { HydrationBoundary, dehydrate } from '@tanstack/react-query'
import { getQueryClient } from '@/lib/get-query-client'
import { TodoList } from './todo-list'

export default async function TodosPage() {
  const queryClient = getQueryClient()

  await queryClient.prefetchQuery({
    queryKey: ['todos'],
    queryFn: fetchTodos
  })

  return (
    <HydrationBoundary state={dehydrate(queryClient)}>
      <TodoList />
    </HydrationBoundary>
  )
}

// app/todos/todo-list.tsx (Client Component)
'use client'

import { useSuspenseQuery } from '@tanstack/react-query'

export function TodoList() {
  const { data } = useSuspenseQuery({
    queryKey: ['todos'],
    queryFn: fetchTodos
  })

  return <ul>
    {data.map(todo => <li key={todo.id}>{todo.title}</li>)}
  </ul>
}
```

**With Suspense boundaries:**

```typescript
export default async function Page() {
  return (
    <Suspense fallback={<TodosSkeleton />}>
      <Todos />
    </Suspense>
  )
}

function Todos() {
  // Suspense shows fallback during loading
  const { data } = useSuspenseQuery({
    queryKey: ['todos'],
    queryFn: fetchTodos
  })

  return <TodoList todos={data} />
}
```

**useQuery vs useSuspenseQuery:**

```typescript
// useQuery
const { data, isLoading } = useQuery(...)
// - data can be undefined
// - Need loading checks
// - Can cause hydration mismatches with SSR

// useSuspenseQuery
const { data } = useSuspenseQuery(...)
// - data is never undefined
// - Throws promise during loading
// - Hydration-safe with SSR
// - Requires Suspense boundary
```

**When NOT to use useSuspenseQuery:**

```typescript
// Client-only data (no SSR prefetch)
const { data } = useQuery({
  queryKey: ['user-preferences'],
  queryFn: fetchUserPreferences
  // ✅ useQuery is fine
})

// Background polling (not SSR critical)
const { data } = useQuery({
  queryKey: ['notifications'],
  queryFn: fetchNotifications,
  refetchInterval: 30000
  // ✅ useQuery is fine
})
```

**References:**
- [Hydration with Prefetch Discussion](https://github.com/TanStack/query/discussions/5487)
- [TanStack Query SSR Guide](https://tanstack.com/query/latest/docs/react/guides/ssr)
