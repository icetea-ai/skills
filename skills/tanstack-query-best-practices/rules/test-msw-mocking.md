---
title: Use Mock Service Worker for Network Mocking
impact: MEDIUM
impactDescription: Consistent API mocking across test environments
tags: testing, msw, mocking, api
---

## Use Mock Service Worker for Network Mocking

Use Mock Service Worker (MSW) for network mocking instead of mocking React Query hooks or fetch directly.

**Why MSW:**

```typescript
// ❌ Mocking the hook
jest.mock('./useTodos')
useTodos.mockReturnValue({ data: mockData })

// Problems:
// - Not testing actual React Query behavior
// - Breaks when hook implementation changes
// - Can't test loading/error states

// ✅ Mock the network layer with MSW
server.use(
  rest.get('/api/todos', (req, res, ctx) => {
    return res(ctx.json(mockData))
  })
)

// Benefits:
// - Tests actual React Query behavior
// - Works with any HTTP client
// - Single source of truth for mocks
```

**Setup:**

```typescript
// test/mocks/server.ts
import { setupServer } from 'msw/node'
import { rest } from 'msw'

export const handlers = [
  rest.get('/api/todos', (req, res, ctx) => {
    return res(
      ctx.json([
        { id: 1, title: 'Test Todo', completed: false },
      ])
    )
  }),
]

export const server = setupServer(...handlers)

// test/setup.ts
import { server } from './mocks/server'

beforeAll(() => server.listen())
afterEach(() => server.resetHandlers())
afterAll(() => server.close())
```

**Basic test:**

```typescript
import { server } from './mocks/server'
import { rest } from 'msw'

test('renders todo list', async () => {
  render(<TodoList />, { wrapper: createWrapper() })

  expect(await screen.findByText('Test Todo')).toBeInTheDocument()
})

test('handles error', async () => {
  server.use(
    rest.get('/api/todos', (req, res, ctx) => {
      return res(ctx.status(500))
    })
  )

  render(<TodoList />, { wrapper: createWrapper() })

  expect(await screen.findByText(/error/i)).toBeInTheDocument()
})
```

**Testing mutations:**

```typescript
test('creates todo', async () => {
  const { result } = renderHook(() => useCreateTodo(), {
    wrapper: createWrapper()
  })

  act(() => {
    result.current.mutate({ title: 'New Todo' })
  })

  await waitFor(() => expect(result.current.isSuccess).toBe(true))
})
```

**Request validation:**

```typescript
test('sends correct payload', async () => {
  let capturedRequest

  server.use(
    rest.post('/api/todos', async (req, res, ctx) => {
      capturedRequest = await req.json()
      return res(ctx.json({ id: 1, ...capturedRequest }))
    })
  )

  const { result } = renderHook(() => useCreateTodo(), {
    wrapper: createWrapper()
  })

  act(() => {
    result.current.mutate({ title: 'Test', priority: 'high' })
  })

  await waitFor(() => expect(result.current.isSuccess).toBe(true))

  expect(capturedRequest).toEqual({
    title: 'Test',
    priority: 'high'
  })
})
```

**Why MSW over alternatives:**

- Works in Node.js, browser, Storybook, Cypress
- Single source of truth for API mocks
- Visible in browser devtools during development
- Supports REST and GraphQL
- Most realistic testing approach

**References:**
- [Testing React Query](https://tkdodo.eu/blog/testing-react-query)
- [MSW Documentation](https://mswjs.io/)
