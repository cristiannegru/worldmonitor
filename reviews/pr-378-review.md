# PR #378 Review: fix: prevent Wingbits API stampede in theater posture

**Reviewer:** Claude (automated)
**Verdict:** Request changes

---

## Summary

The PR correctly identifies and addresses a real thundering-herd problem: when the
primary cache expires, every concurrent request independently fires
`POST /v1/flights` to Wingbits with 9 bounding boxes, causing ~1,500 requests/min
during the ~15s refill window.

The fix leverages `cachedFetchJson` which uses an in-flight promise map to coalesce
concurrent cache misses into a single upstream fetch. The approach is sound and
consistent with the pattern already used in the trade endpoints.

---

## Bug: Missing error handling breaks the fallback chain (Critical)

When both OpenSky and Wingbits are unavailable, `fetchTheaterPostureFresh` throws.
This rejection propagates through `cachedFetchJson` (which doesn't catch fetcher
errors) and hits the unguarded `await` in `getTheaterPosture`:

```typescript
// Line 220-224 — no try-catch around this await
const result = await cachedFetchJson<GetTheaterPostureResponse>(
  CACHE_KEY,
  CACHE_TTL,
  fetchTheaterPostureFresh,   // throws when both sources fail
);
if (result) return result;    // ← never reached on rejection

// Lines 228-232 — fallback code is unreachable on error
const stale = ...
```

The **old code** wrapped the fetch in a `try-catch` (lines 193-224) that gracefully
fell through to stale/backup tiers. The new code lost this safety net. When both
upstreams are down, instead of returning stale data, the handler will throw an
unhandled rejection to the caller.

### Fix — either:

**(a)** Wrap the `cachedFetchJson` call in try-catch:

```typescript
let result: GetTheaterPostureResponse | null = null;
try {
  result = await cachedFetchJson<GetTheaterPostureResponse>(
    CACHE_KEY, CACHE_TTL, fetchTheaterPostureFresh,
  );
} catch { /* fall through to stale/backup */ }
if (result) return result;
```

**(b)** Or make the fetcher return `null` instead of throwing:

```typescript
async function fetchTheaterPostureFresh(): Promise<GetTheaterPostureResponse | null> {
  // ...
  } else {
    return null;  // instead of: throw new Error(...)
  }
}
```

Option (b) is cleaner — it aligns with how `cachedFetchJson` is used in the trade
endpoints (`get-trade-flows.ts`, `get-trade-restrictions.ts`) where the fetcher
returns `null` on failure, and `cachedFetchJson` correctly skips caching null values.

---

## Concern: Fire-and-forget stale/backup writes may not complete (Medium)

```typescript
// Lines 206-209 — not awaited
Promise.all([
  setCachedJson(STALE_CACHE_KEY, result, STALE_TTL),
  setCachedJson(BACKUP_CACHE_KEY, result, BACKUP_TTL),
]).catch(() => {});
```

The old code `await`ed all three cache writes before returning. The new code fires
off stale/backup writes in the background. In an Edge Function runtime, once the
response is sent, the runtime may terminate the isolate — these writes can be dropped
silently.

This degrades the multi-tier resilience that's the safety net for upstream outages.
If stale/backup tiers never get refreshed, a prolonged upstream outage after the
stale data expires will return `{ theaters: [] }` instead of day-old or week-old data.

### Suggestion

Await the stale/backup writes inside the fetcher. The extra ~3ms for two Redis SET
calls is negligible compared to the upstream fetch time:

```typescript
const result: GetTheaterPostureResponse = { theaters };
await Promise.all([
  setCachedJson(STALE_CACHE_KEY, result, STALE_TTL),
  setCachedJson(BACKUP_CACHE_KEY, result, BACKUP_TTL),
]);
return result;
```

---

## Minor: No tests for the changed handler (Low)

There are no existing tests for `getTheaterPosture`, and the PR doesn't add any.
Given this fixes a production incident, a test verifying that concurrent calls result
in exactly one upstream fetch would provide confidence against regression.

The test infrastructure already supports this pattern — see the existing
`cachedFetchJson` coalescing test in `tests/redis-caching.test.mjs:69-118`.

---

## What's good

- The stampede diagnosis is accurate and well-documented in the PR description
- `cachedFetchJson` is the right primitive for this — reuses existing infra
- Consistent with the pattern in `get-trade-flows.ts` and `get-trade-restrictions.ts`
- The manual primary cache check was removed since `cachedFetchJson` handles it internally
- Parallel source racing (OpenSky + Wingbits via `Promise.allSettled`) is preserved
