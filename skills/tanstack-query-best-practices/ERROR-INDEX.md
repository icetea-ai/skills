# Common Errors → Rule Reference

Quick lookup for common React Query errors and which rule file solves them.

## Error Messages

### "Property 'onSuccess' does not exist"
**Issue:** Query callbacks removed from useQuery in v5
**Fix:** See [mutation-prefer-mutate.md](./rules/mutation-prefer-mutate.md)
**Note:** Still available in useMutation

### "Property 'cacheTime' does not exist"
**Issue:** Renamed to `gcTime` in v5
**Fix:** See [data-stale-vs-gc.md](./rules/data-stale-vs-gc.md)

### "Type 'string[]' is not assignable to type 'QueryKey'"
**Issue:** Query keys must be arrays
**Fix:** See [keys-use-arrays.md](./rules/keys-use-arrays.md)

### "Property 'enabled' does not exist on type 'UseSuspenseQueryOptions'"
**Issue:** Cannot use `enabled` with useSuspenseQuery
**Fix:** See [ts-skiptoken-dependent.md](./rules/ts-skiptoken-dependent.md)

### "Hydration failed because the initial UI does not match"
**Issue:** Server/client render mismatch
**Fixes:**
- [ssr-suspense-query.md](./rules/ssr-suspense-query.md)
- [ssr-streaming-hydration.md](./rules/ssr-streaming-hydration.md)
- [ssr-no-fetchstatus-suspense.md](./rules/ssr-no-fetchstatus-suspense.md)

### "Text content does not match server-rendered HTML"
**Issue:** Hydration error with prefetched data
**Fix:** See [ssr-suspense-query.md](./rules/ssr-suspense-query.md)

## Common Problems

### Query never refetches / Data always stale
**Possible causes:**
- staleTime too high → [perf-global-stale-time.md](./rules/perf-global-stale-time.md)
- Incorrect staleTime vs gcTime → [data-stale-vs-gc.md](./rules/data-stale-vs-gc.md)
- Query key not updating → [keys-as-dependencies.md](./rules/keys-as-dependencies.md)

### Mutation succeeds but UI doesn't update
**Possible causes:**
- Not invalidating queries → [mutation-invalidate-vs-update.md](./rules/mutation-invalidate-vs-update.md)
- Not awaiting invalidation → [mutation-await-invalidate.md](./rules/mutation-await-invalidate.md)

### Can't access specific error properties (AxiosError, etc.)
**Issue:** Error type narrowing
**Fix:** See [ts-error-type-narrowing.md](./rules/ts-error-type-narrowing.md)

### Component re-renders on every background refetch
**Possible causes:**
- Not using tracked queries → [perf-tracked-queries.md](./rules/perf-tracked-queries.md)
- Accessing too many properties → [perf-select-memoization.md](./rules/perf-select-memoization.md)

### Syncing query data to useState causes bugs
**Issue:** Dual source of truth
**Fix:** See [data-no-sync-state.md](./rules/data-no-sync-state.md)

### Derived state out of sync with query data
**Issue:** Using useEffect to derive state
**Fix:** See [data-derive-during-render.md](./rules/data-derive-during-render.md)

### Tests timeout or fail randomly
**Possible causes:**
- Retries enabled → [test-disable-retries.md](./rules/test-disable-retries.md)
- Shared QueryClient → [test-unique-client.md](./rules/test-unique-client.md)

### TypeScript errors with dependent queries
**Issue:** Parameter might be undefined
**Fix:** See [ts-skiptoken-dependent.md](./rules/ts-skiptoken-dependent.md)

### Type inference not working
**Issue:** Manually specifying generics
**Fix:** See [ts-infer-from-queryfn.md](./rules/ts-infer-from-queryfn.md)

### Optimistic update doesn't rollback on error
**Issue:** Missing rollback logic
**Fix:** See [mutation-optimistic-rollback.md](./rules/mutation-optimistic-rollback.md)

## Not Covered (Advanced Topics)

The following are mentioned in jezweb's skill but not yet in this skill:

- `initialPageParam` required (useInfiniteQuery)
- `keepPreviousData` → `placeholderData: keepPreviousData`
- `isLoading` vs `isPending` change in v5
- `useErrorBoundary` → `throwOnError` rename

For these, refer to [TanStack Query v5 Migration Guide](https://tanstack.com/query/latest/docs/react/guides/migrating-to-v5).

## Quick Wins

**Most common fixes:**
1. Use array keys → [keys-use-arrays.md](./rules/keys-use-arrays.md)
2. Set global staleTime → [perf-global-stale-time.md](./rules/perf-global-stale-time.md)
3. Never sync to useState → [data-no-sync-state.md](./rules/data-no-sync-state.md)
4. Use useSuspenseQuery for SSR → [ssr-suspense-query.md](./rules/ssr-suspense-query.md)
5. Disable retries in tests → [test-disable-retries.md](./rules/test-disable-retries.md)
