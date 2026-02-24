---
title: Treat Query Keys Like Dependency Arrays
impact: CRITICAL
impactDescription: Prevents stale closures and enables automatic refetching
tags: query-keys, dependencies, refetching, declarative
---

## Treat Query Keys Like Dependency Arrays

Include all variables used in `queryFn` in the query key - queries are declarative, not imperative.

**Incorrect (imperative refetching):**

```typescript
const [filters, setFilters] = useState('all')

// ❌ Query key doesn't include filters
const { data, refetch } = useQuery({
  queryKey: ['todos'],
  queryFn: () => fetchTodos(filters) // Stale closure!
})

// ❌ Trying to pass parameters to refetch
return <Filters onApply={(newFilters) => {
  setFilters(newFilters)
  refetch(???) // Can't pass parameters
}} />
```

**Correct (declarative dependencies):**

```typescript
const [filters, setFilters] = useState('all')

// ✅ Query key includes all queryFn dependencies
const { data } = useQuery({
  queryKey: ['todos', filters],
  queryFn: () => fetchTodos(filters) // Always uses current filters
})

// ✅ Setting state automatically triggers refetch
return <Filters onApply={setFilters} />
```

**Why this matters:**

When `filters` changes:
1. React Query sees the key changed
2. Automatically fetches with new filters
3. No manual refetch needed
4. No stale closure bugs

**Think declaratively:**

```typescript
// ❌ Imperative: "fetch this, then refetch when I tell you"
refetch()

// ✅ Declarative: "fetch based on this state"
queryKey: ['todos', filters, sortBy, page]
```

**Rule of thumb:** If your `queryFn` uses a variable, that variable must be in the `queryKey`.

**References:**
- [Effective React Query Keys](https://tkdodo.eu/blog/effective-react-query-keys)
- [Practical React Query](https://tkdodo.eu/blog/practical-react-query)
