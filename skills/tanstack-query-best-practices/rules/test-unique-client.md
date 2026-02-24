---
title: Create New QueryClient Per Test
impact: CRITICAL
impactDescription: Prevents test pollution and flaky tests
tags: testing, isolation, test-setup
---

## Create New QueryClient Per Test

Create a fresh `QueryClient` for every test case to ensure complete isolation.

**Incorrect (shared client):**

```typescript
// ❌ Shared across all tests
const queryClient = new QueryClient()

test('fetches todos', async () => {
  // Uses shared client
})

test('fetches user', async () => {
  // Still has todos in cache from previous test!
})
```

**Correct (isolated clients):**

```typescript
const createWrapper = () => {
  const queryClient = new QueryClient({
    defaultOptions: {
      queries: { retry: false },
    },
  })

  return ({ children }) => (
    <QueryClientProvider client={queryClient}>
      {children}
    </QueryClientProvider>
  )
}

test('fetches todos', async () => {
  const { result } = renderHook(() => useTodos(), {
    wrapper: createWrapper() // New client
  })

  await waitFor(() => expect(result.current.isSuccess).toBe(true))
})
```

**Why this matters:**

Without isolation:
- Test 1 adds data to cache
- Test 2 expects empty cache but finds test 1's data
- Flaky failures in parallel execution
- Order-dependent tests

With new client per test:
- Each test starts with clean cache
- Tests can run in any order
- No shared state between tests
- Reliable parallel execution

**Reusable test utility:**

```typescript
// test-utils.tsx
export function createTestQueryClient() {
  return new QueryClient({
    defaultOptions: {
      queries: { retry: false, gcTime: Infinity },
      mutations: { retry: false },
    },
  })
}

export function createWrapper() {
  const queryClient = createTestQueryClient()
  return ({ children }) => (
    <QueryClientProvider client={queryClient}>
      {children}
    </QueryClientProvider>
  )
}
```

**References:**
- [Testing React Query](https://tkdodo.eu/blog/testing-react-query)
