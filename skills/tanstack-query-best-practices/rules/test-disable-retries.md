---
title: Disable Retries in Test Environment
impact: CRITICAL
impactDescription: Prevents test timeouts from retry delays
tags: testing, configuration, retries, timeouts
---

## Disable Retries in Test Environment

Set `retry: false` in test QueryClient configuration to prevent timeouts from React Query's default retry logic.

**The problem:**

```typescript
// ❌ Default configuration in tests
const queryClient = new QueryClient()

test('handles fetch error', async () => {
  // React Query retries 3 times with exponential backoff
  // Total: ~3+ seconds before test completes
  // Often causes test timeout!
})
```

**Solution:**

```typescript
// ✅ Disable retries in tests
const queryClient = new QueryClient({
  defaultOptions: {
    queries: { retry: false },
    mutations: { retry: false },
  },
})
```

**Why this matters:**

Default retry behavior:
- Retry 1: immediate
- Retry 2: 1 second delay
- Retry 3: 2 second delay
- Total: 3+ seconds per failed request
- Test timeouts and slow test suites

With `retry: false`:
- Errors fail immediately
- Fast test execution
- No timeout issues

**Gotcha - Query-level config overrides defaults:**

```typescript
// Global test config
const queryClient = new QueryClient({
  defaultOptions: {
    queries: { retry: false },
  },
})

// In your hook
function useTodos() {
  return useQuery({
    queryKey: ['todos'],
    queryFn: fetchTodos,
    retry: 3, // ❌ Overrides test default!
  })
}
```

**Solution - Use setQueryDefaults:**

```typescript
function useTodos() {
  const queryClient = useQueryClient()

  // Production default
  queryClient.setQueryDefaults(['todos'], {
    retry: 3,
  })

  return useQuery({
    queryKey: ['todos'],
    queryFn: fetchTodos,
    // No inline retry - allows test override
  })
}
```

**Complete test setup:**

```typescript
// test-utils.tsx
export function createTestQueryClient() {
  return new QueryClient({
    defaultOptions: {
      queries: {
        retry: false,
        gcTime: Infinity,
      },
      mutations: {
        retry: false,
      },
    },
    logger: {
      log: console.log,
      warn: console.warn,
      error: () => {}, // Silence expected errors
    },
  })
}
```

**References:**
- [Testing React Query](https://tkdodo.eu/blog/testing-react-query)
