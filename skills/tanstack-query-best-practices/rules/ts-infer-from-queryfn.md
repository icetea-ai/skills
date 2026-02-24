---
title: Let TypeScript Infer Types from queryFn
impact: MEDIUM
impactDescription: Type-safe queries without manual generic specifications
tags: typescript, type-inference, type-safety
---

## Let TypeScript Infer Types from queryFn

Let TypeScript infer query types from the `queryFn` return type instead of manually specifying generics.

**Incorrect (manual generics):**

```typescript
// ❌ Verbose and error-prone
useQuery<Todo[], Error, Todo[], string[]>({
  queryKey: ['todos'],
  queryFn: fetchTodos
})

// Problems:
// - Must specify all 4 generics
// - Easy to get wrong
// - Hard to maintain
```

**Correct (type inference):**

```typescript
// ✅ Type the queryFn return value
function fetchTodos(): Promise<Todo[]> {
  return axios.get('/todos').then(res => res.data)
}

const { data } = useQuery({
  queryKey: ['todos'],
  queryFn: fetchTodos
})
// data is automatically Todo[] | undefined

// ✅ Or inline with type assertion
const { data } = useQuery({
  queryKey: ['todos'],
  queryFn: async () => {
    const res = await axios.get<Todo[]>('/todos')
    return res.data
  }
})
```

**With select (automatic inference):**

```typescript
const { data } = useQuery({
  queryKey: ['todos'],
  queryFn: fetchTodos, // Returns Promise<Todo[]>
  select: (data) => data.map(t => t.name)
  // data param: Todo[]
  // returned data: string[]
})
// Final type: string[] | undefined
```

**Custom hooks with inference:**

```typescript
function useTodos() {
  return useQuery({
    queryKey: ['todos'],
    queryFn: fetchTodos // Returns Promise<Todo[]>
  })
}

const { data } = useTodos()
// data is Todo[] | undefined (inferred)
```

**Why this works better:**

```typescript
// Single source of truth
async function fetchUser(): Promise<User> {
  return api.get('/user').then(r => r.data)
}

// Type flows automatically:
// fetchUser → Promise<User>
// useQuery infers → User | undefined
// select infers → (data: User) => ...

const { data } = useQuery({
  queryKey: ['user'],
  queryFn: fetchUser,
  select: (user) => user.name // user: User
})
// data: string | undefined
```

**Error type inference:**

```typescript
// Default (v5)
const { error } = useQuery({
  queryKey: ['todos'],
  queryFn: fetchTodos
})
// error: Error | null

// Custom via module augmentation
declare module '@tanstack/react-query' {
  interface Register {
    defaultError: AxiosError
  }
}

// Now all queries use AxiosError
const { error } = useQuery(...)
// error: AxiosError | null
```

**Type narrowing with status:**

```typescript
const query = useQuery({
  queryKey: ['todo'],
  queryFn: fetchTodo
})

// ✅ Keep object reference for narrowing
if (query.isSuccess) {
  // query.data is Todo (narrowed)
}

// ✅ TypeScript 4.6+ supports destructured narrowing
const { data, isSuccess } = query
if (isSuccess) {
  // data is Todo (narrowed in TS 4.6+)
}
```

**References:**
- [React Query and TypeScript](https://tkdodo.eu/blog/react-query-and-type-script)
- [Type-safe React Query](https://tkdodo.eu/blog/type-safe-react-query)
