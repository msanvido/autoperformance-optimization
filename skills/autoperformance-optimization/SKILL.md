---
name: autoperformance-optimization
description: >
  Systematic performance optimization skill that benchmarks existing code, identifies improvements,
  implements and compares solutions, and iterates to find the best approach.
  **Triggers:** "optimize performance", "speed up", "make it faster", "performance improvement",
  "benchmark and improve", "this is too slow", "reduce load time", "improve response time",
  or any request to measure and improve the speed of an API, page, or task.
---

# Auto Performance Optimization

A systematic, data-driven approach to improving the performance of APIs, pages, or tasks.
The process follows a strict measure-first, iterate-often discipline: never optimize without
a benchmark, never ship without comparing against the baseline.

## Process Overview

```
Phase 1: Define & Baseline   -> Identify the slow path, build benchmarks
Phase 2: Identify Solutions   -> Propose 3-5 optimization strategies
Phase 3: Implement & Compare  -> Build each, measure against baseline
Phase 4: Iterate Winners      -> Take the best 2-3, iterate 3x each
Phase 5: Validate & Ship      -> Final benchmark, pick the winner
```

---

## Phase 1: Define the Target & Build Benchmarks

### 1.1 -- Define the Target Scope

Ask the user to specify the exact scope of the optimization: which API endpoint(s), page route(s), or background task(s) to focus on. Do not proceed until the target is clearly defined.

### 1.2 -- Identify the Slow Path

Before touching any code, clearly define what is slow and how users experience it.

- **What to capture:**
  - The specific API endpoint(s), page route(s), or background task(s)
  - The user-facing symptom (e.g., "page takes 20s to load", "API returns in 8s")
  - The data volume involved (e.g., "718 activities across 6 types over 30 days")
  - Current architecture: how data flows from DB -> backend -> frontend -> user

- **How to investigate:**
  - Read the relevant backend view/serializer and frontend loader/component
  - Check for N+1 queries, missing pagination, unnecessary serialization, redundant auth calls
  - Use browser DevTools Network tab or `time curl` to get baseline numbers
  - Check if the bottleneck is backend (query time), network (payload size), or frontend (render time)

### 1.3 -- Build Benchmarks

Create automated benchmarks that measure performance under multiple conditions.
Benchmarks must be **repeatable**, **automated**, and cover **realistic scenarios**.

**What to benchmark (at minimum):**

| Scenario | Why |
|----------|-----|
| Typical load (default parameters) | The common case users hit daily |
| Heavy load (max data range / large dataset) | Worst-case performance envelope |
| Targeted operation (e.g., filter by type, single entity) | Measures focused queries vs full scans |
| Sequential rapid operations (e.g., click A then immediately B) | Measures request cancellation / queuing |
| Full completion (load everything) | Total data integrity after all pages load |

**Implementation approach:**

- **For web pages:** Write Playwright or Cypress E2E tests with timing assertions
  - Measure wall-clock time from navigation start to first meaningful content
  - Use `Date.now()` or `performance.now()` before/after key operations
  - Assert hard time budgets (e.g., `expect(elapsed).toBeLessThan(10_000)`)
  - Test both initial load AND subsequent interactions (scroll, filter, navigate)

- **For APIs:** Write integration tests or scripts that call endpoints directly
  - Measure response time with `time curl` or programmatic timing
  - Test with varying payload sizes and query parameters
  - Measure both cold-start and warm/cached performance

- **For background tasks:** Measure task duration via Celery/job logs
  - Compare processing time for small vs large batches

**Example benchmark structure (from activity digest work):**

```
Test 1: Initial load completes in under 10 seconds
Test 2: Jump-to loads a targeted type in under 3 seconds
Test 3: Rapid click A -> click B aborts A, B loads fast
Test 4: Infinite scroll loads additional pages
Test 5: Full scroll loads all data to completion
Test 6: Data counts match between nav, headers, and rendered items
```

Save the baseline numbers in a markdown table -- you will compare against these throughout.

---

## Phase 2: Identify Optimization Strategies

Propose **at least 3, ideally 5** distinct optimization strategies. Consider these categories:

### Common optimization patterns

1. **Server-side pagination** -- Return data in pages (30-50 items) instead of loading everything at once. Add `page`, `page_size` params to the API. Return `type_totals` or `total_count` via a cheap COUNT/GROUP BY query so the frontend knows totals without loading all data.

2. **Concurrent / parallel fetching** -- Replace serialized sequential requests with concurrent `fetch()` + `AbortController`. When the user clicks a new target, abort the previous in-flight request immediately.

3. **Auth optimization** -- Skip redundant authentication on subsequent requests (pagination, sub-fetches) when the user was already validated on the initial load. Use `validate: false` or equivalent.

4. **Query optimization** -- Eliminate N+1 queries with `select_related`/`prefetch_related`, add DB indexes, use `only()`/`defer()` to reduce column fetches, replace ORM with raw SQL for hot paths.

5. **Frontend rendering optimization** -- Virtual scrolling for large lists, `React.memo` for expensive components, `IntersectionObserver` for lazy loading, debounced search inputs.

6. **Caching** -- Redis/memcached for expensive queries, HTTP cache headers for static-ish data, client-side cache with stale-while-revalidate.

7. **Payload reduction** -- Return only needed fields, compress responses, use efficient serialization (ORJSON vs standard JSON).

8. **Dedicated resource routes** -- Create lightweight endpoints for specific sub-operations (e.g., a jump-to route) that bypass the full page loader machinery.

### How to prioritize

For each strategy, estimate:
- **Impact**: How much time will this save? (High/Medium/Low)
- **Effort**: How complex is the implementation? (High/Medium/Low)
- **Risk**: Could this break existing functionality? (High/Medium/Low)

Start with high-impact, low-effort strategies.

### 2.2 -- Ask the user which strategies to implement

Present the proposed strategies with their impact/effort/risk estimates to the user. Ask them to confirm which strategies should be implemented in Phase 3. Do not proceed until the user confirms.

---

## Phase 3: Implement & Measure Each Strategy

### 3.1 -- Implement one strategy at a time

- Create a separate git branch for each strategy (e.g., `perf/strategy-1-pagination`, `perf/strategy-2-parallel-fetch`)
- Make a clean, isolated change for each strategy
- Commit each independently so you can measure and compare
- Keep the benchmark tests passing throughout

### 3.2 -- Measure against baseline

After implementing each strategy, run the full benchmark suite and record results:

```markdown
| Scenario              | Baseline | Strategy 1 | Strategy 2 | Strategy 3 |
|-----------------------|----------|------------|------------|------------|
| Initial load          | 20s      | 2.5s       | 18s        | 15s        |
| Targeted operation    | 20.8s    | 500ms      | 19s        | 12s        |
| Rapid cancel + reload | 6.7s     | 500ms      | 5s         | 6s         |
| Full completion       | 20s      | 40s (paginated) | 18s   | 15s        |
```

### 3.3 -- Ask the user which strategies to iterate on

Present the benchmark comparison table to the user and ask them to choose which strategies should move to Phase 4 for further iteration. Do not proceed until the user confirms.

### 3.4 -- Identify winners and losers

- Any strategy that shows < 10% improvement: **drop it** (not worth the complexity)
- Any strategy that regresses a scenario: **investigate** before proceeding
- The top 2-3 strategies (as confirmed by the user) move to Phase 4

---

## Phase 4: Iterate on Winners (3 iterations each)

For each winning strategy, do **at least 3 iteration rounds**:

### Iteration 1: Refine the implementation
- Fix edge cases found during benchmarking
- Example: Stacked queryset slices in Django -> replace with single bounded slice
- Example: Section header counts showing inflated DB totals -> switch to actual rendered count

### Iteration 2: Optimize further
- Look for secondary gains within the same strategy
- Example: After adding pagination, also add `type_totals` GROUP BY to avoid loading all data just for counts
- Example: After adding AbortController, also add a dedicated resource route to avoid full page loader overhead

### Iteration 3: Polish and harden
- Handle edge cases: empty states, error recovery, SSR compatibility
- Example: Hide back-to-top button when at top of page
- Example: Add fallback for broken avatar images
- Example: Sync nav counts with actual rendered counts after full load

**After each iteration, re-run the full benchmark suite and update the comparison table.**

### 4.4 -- Compare strategies and ask the user to choose

Present a final comparison of all iterated strategies with their benchmark results side by side. Ask the user to select the winning strategy. Do not proceed until the user confirms.

Once the user chooses, merge the winning strategy's branch on top of the current baseline branch. Phase 5 continues on this merged branch (i.e., the original code with the chosen strategy's changes applied).

---

## Phase 5: Validate & Ship

### 5.1 -- Final benchmark

Run the complete benchmark suite one final time with all improvements combined:

```markdown
| Scenario                    | Before  | After   | Improvement |
|-----------------------------|---------|---------|-------------|
| Initial page load           | ~20s    | ~2.5s   | 8x faster   |
| Jump to small type (2 items)| blocked | ~500ms  | instant     |
| Jump to large type (639)    | 20.8s   | ~2.7s   | 7.7x faster |
| Cancel + re-navigate        | ~6.7s   | ~500ms  | 13x faster  |
| Auth overhead per request   | ~60ms   | 0ms     | eliminated  |
```

### 5.2 -- Verify data integrity

Performance improvements must not sacrifice correctness:
- Total item counts must match between backend totals, frontend nav, and rendered items
- All data types must appear (watch for silently dropped non-standard types)
- Search, filter, and date range features must still work correctly with pagination
- Test with edge cases: empty date ranges, single-item types, very large datasets

### 5.3 -- Write the E2E performance tests

The benchmark tests from Phase 1 become permanent E2E tests that prevent regressions:
- Assert time budgets (e.g., initial load < 10s)
- Assert data completeness (all types present, counts match)
- Assert UX behaviors (infinite scroll works, cancel+re-navigate works)
- These tests should run against a real backend, not mocks, for realistic timing

### 5.4 -- Write a performance report and document the results

Produce a standalone performance report (markdown file in the repo or PR body) that covers:

- **All optimizations implemented** -- what was done, why it was chosen, and the measured impact of each
- **Strategies considered but not taken** -- what was evaluated and dropped, with the reasoning (e.g., < 10% gain, too risky, high complexity for marginal benefit)
- **Before/after comparison table** -- clearly show baseline vs final numbers for every benchmark scenario
- **Tests and benchmarks run** -- list every test/benchmark executed, what it measures, and the pass/fail results
- **Summary** -- headline improvement (e.g., "8x faster initial load") and any caveats or follow-up work

This report should be self-contained so that anyone reviewing the PR can understand the full optimization journey without reading the code. Also include a summary table in the PR description showing before/after for every scenario with the specific optimizations applied and their individual contributions.

---

## Anti-patterns to Avoid

- **Optimizing without measuring first** -- You will not know if you made things better or worse
- **Measuring only the happy path** -- Test edge cases, large datasets, and rapid interactions
- **Combining multiple changes before measuring** -- You cannot attribute gains to specific strategies
- **Dropping a strategy after one attempt** -- Iterate at least 3 times before giving up on a promising approach
- **Breaking correctness for speed** -- Always verify data integrity after optimization
- **Over-caching** -- Cache invalidation bugs are worse than slow loads; only cache when the data lifecycle is well-understood
- **Premature abstraction** -- Don't build a generic pagination framework; just paginate the slow endpoint

---

## Checklist

Use this checklist to track progress through the optimization process:

```
Phase 1: Define & Baseline
[ ] User specified the target scope (endpoint, page, or task)
[ ] Identified the slow path and user-facing symptom
[ ] Measured baseline performance numbers
[ ] Built automated benchmark tests (at least 5 scenarios)
[ ] Recorded baseline numbers in a comparison table

Phase 2: Identify Solutions
[ ] Proposed at least 3 distinct optimization strategies
[ ] Estimated impact/effort/risk for each
[ ] Prioritized strategies by expected ROI

Phase 3: Implement & Compare
[ ] Implemented each strategy in isolation
[ ] Ran benchmarks after each strategy
[ ] Updated comparison table with results
[ ] Identified top 2-3 winners

Phase 4: Iterate Winners (3x each)
[ ] Iteration 1: Refined implementation, fixed edge cases
[ ] Iteration 2: Found secondary optimization gains
[ ] Iteration 3: Polished, hardened, handled edge cases
[ ] Re-ran benchmarks after each iteration

Phase 5: Validate & Ship
[ ] Final benchmark shows clear improvement
[ ] Data integrity verified (counts match, no dropped data)
[ ] E2E performance tests written and passing
[ ] Performance report written (optimizations done, not taken, improvements, benchmarks)
[ ] PR description includes before/after comparison table
```
