# Autoperformance Optimization: Letting Claude Code Be Your Performance Engineer

Karpathy's [autoresearch](https://github.com/karpathy/autoresearch) showed that you can point an AI agent at a research question and let it systematically explore the space -- reading papers, forming hypotheses, running experiments, and iterating on what works. I wanted to apply the same idea to a different domain: **performance optimization**.

The result is **autoperformance-optimization**, a Claude Code skill that turns performance tuning from a manual, intuition-driven process into a systematic, benchmark-driven loop. You point it at a slow endpoint or page, and it does the rest: measures a baseline, proposes strategies, implements them one at a time, benchmarks each against the original, iterates on the winners, and produces a final report with before/after numbers.

This post walks through two real-world examples where this skill delivered 8-11x performance improvements on production systems.

## The approach

The core loop is simple:

```
Measure -> Hypothesize -> Implement -> Measure -> Iterate
```

What makes it work as a skill (rather than a one-off prompt) is the discipline it enforces:

1. **Never optimize without a baseline.** The skill's first step is always to build automated benchmarks -- Playwright E2E tests for pages, timed API calls for endpoints -- and record baseline numbers in a table. Every subsequent change is measured against this table.

2. **Propose multiple strategies, implement them in isolation.** Instead of jumping to the first optimization idea, the skill proposes 3-5 distinct strategies (pagination, caching, query optimization, parallel fetching, etc.), estimates impact/effort/risk for each, and asks you to choose which to pursue. Each strategy gets its own branch and its own benchmark run.

3. **Iterate at least 3 times on winners.** A single implementation pass rarely captures the full gain. The skill mandates three iteration rounds: refine edge cases, find secondary gains within the same strategy, then polish and harden.

4. **Verify correctness after every change.** Performance improvements that break data integrity are worse than being slow. The skill checks that counts match, data types are preserved, and filters still work after every optimization.

## Two important caveats

Before diving into the examples, two lessons learned the hard way:

### Avoid premature optimization

This skill works best on code paths that have **stabilized** in the codebase. If the feature is still being actively developed -- if the schema is changing, if requirements are shifting, if the endpoint's contract isn't settled -- then optimizing it is a waste of effort. You'll spend time tuning queries for a data model that changes next week, or caching responses whose shape is about to be reworked.

Wait until the code path has settled. When a page or endpoint has been in production for a while, its usage patterns are clear, and its interface is stable -- that's when optimization pays off. The activity digest and inbox dashboard examples below were both mature features that had been live for months before we pointed the skill at them.

### Re-benchmark after every corner case fix

This one is subtle and easy to miss. During Phase 4 (iterating on winners), you'll inevitably discover edge cases: non-standard data types being silently dropped, section header counts not matching rendered counts, stacked queryset slices producing incorrect results.

Fixing these corner cases is necessary for correctness. But here's the trap: **the fix for a corner case can undo the performance gain on the common path.** Or worse, it can make the less-traveled code path dramatically slower while the happy path stays fast -- and your benchmark suite, if it only tests the happy path, won't catch it.

The skill addresses this by requiring a full benchmark re-run after every iteration, not just the final one. But the human lesson is: when you see a corner case fix go in, be skeptical. Re-run the benchmarks. Check the scenarios you don't normally test. The common path is now fast, but the less-beaten path could become slow.

## Example 1: Activity digest -- 8x faster initial load

**The problem:** The activity digest page (`/utils/activity-digest`) loaded 30 days of data in a single request. With 718 activities across 6 types, the page took **~20 seconds** to become interactive. The entire page was blocked -- no progressive rendering, no ability to navigate to a specific activity type without waiting for everything.

**Phase 1 -- Baseline benchmarks (Playwright E2E):**

| Scenario | Baseline |
|----------|----------|
| Initial page load (all data) | ~20s |
| Jump to small type (UBM, 2 items) | Blocked behind full load |
| Jump to large type (Meeting Notes, 639 items) | 20.8s |
| Click type A, immediately click type B | B waits ~6.7s for A to finish |
| Auth overhead per pagination request | ~60ms (`/users/me/` call) |

**Phase 2 -- Strategies proposed:**

The skill identified five strategies:
1. Server-side pagination (30 items per page, GROUP BY for totals)
2. Concurrent jump-to navigation with AbortController
3. Auth optimization (skip re-validation on pagination requests)
4. Dedicated resource route for jump-to fetches
5. IntersectionObserver-based infinite scroll

All five were approved for implementation.

**Phase 3 -- Implementation & measurement:**

Each strategy was implemented in isolation and benchmarked. Server-side pagination had the largest single impact (20s -> 2.5s for initial load). AbortController-based navigation eliminated the "click A then B" blocking problem. Auth optimization removed 60ms per request. The dedicated resource route bypassed React Router's `useFetcher` serialization for jump-to fetches.

**Phase 4 -- Iterations:**

Three rounds of iteration caught and fixed:
- Non-enum activity types (like "UBM") being silently dropped from results
- Section header counts showing inflated database totals instead of actual rendered counts
- `hasMore` state contamination from jump pages affecting the main scroll
- Page number skip on network errors during infinite scroll

Each fix was followed by a full benchmark re-run to confirm no regressions.

**Phase 5 -- Final results:**

| Scenario | Before | After | Improvement |
|----------|--------|-------|-------------|
| Initial page load | ~20s | ~2.5s | 8x faster |
| Jump to UBM (2 items) | Blocked | ~500ms | Instant |
| Jump to Meeting Notes (639 items) | 20.8s | ~2.7s | 7.7x faster |
| Click Onboarding -> immediately click UBM | UBM waited ~6.7s | ~500ms | 13x faster |
| Auth overhead per request | ~60ms | 0ms | Eliminated |

The PR shipped with 9 Playwright integration tests that verify load time, jump-to abort behavior, infinite scroll completion, and count accuracy -- permanent regression prevention.

## Example 2: Inbox dashboard API -- 11x faster warm responses

**The problem:** The inbox dashboard endpoint (`/utils/inbox-dashboard`) aggregated data from multiple sources: IntroRequest queries for each user, Airtable fetches for opportunities/votes/investors, and company domicile lookups. The result was a **~6 second** response time on warm requests and **~11 seconds** cold. For an internal dashboard that people check multiple times a day, this was painful.

**Phase 1 -- Baseline benchmarks:**

| Scenario | Baseline |
|----------|----------|
| Initial load (table visible in browser) | 6,360ms |
| Page refresh | 5,890ms |
| Direct API response time | 5,547ms |

The skill also built a Django management command (`benchmark_inbox_dashboard`) that could profile individual sections of the response to identify where time was being spent.

**Phase 2 -- Strategies proposed:**

The skill identified four main strategies:
1. Query consolidation (12 per-user IntroRequest query loops -> 2 bulk queries)
2. Parallel external fetches (Airtable calls via ThreadPoolExecutor)
3. Response-level Redis caching (60s TTL)
4. Batch company domicile lookups (N individual Airtable calls -> 1 batch call)

**Phase 3 -- Implementation & measurement:**

Query consolidation delivered the largest gain -- the N+1 loop of individual IntroRequest queries was the dominant bottleneck. Parallelizing the Airtable fetches with `ThreadPoolExecutor` reduced the serial chain of external API calls. Redis caching made subsequent requests near-instant. Batch domicile lookups eliminated another N individual Airtable calls.

**Phase 4 -- Iterations:**

Iteration rounds uncovered:
- A `timezone.utc` vs `timezone.UTC` bug (Python 3.14 compatibility) that silently broke "Missing Investment Thesis" counts
- Celery Redis SSL configuration not handling `rediss://` URLs correctly
- Pending vote computation that could be moved to a background thread, running concurrently with other data fetching

The timezone bug is a good example of why correctness checks matter: the optimization work exposed a pre-existing bug that had been silently producing wrong results. The benchmark's data integrity checks caught it.

**Phase 5 -- Final results:**

| Scenario | Before | After | Improvement |
|----------|--------|-------|-------------|
| Initial load (table visible) | 6,360ms | 1,739ms | 3.7x faster |
| Page refresh | 5,890ms | 257ms | 22.9x faster |
| Direct API response | 5,547ms | 503ms | 11x faster |

The PR shipped with both a Django benchmark management command for backend profiling and Playwright E2E tests for end-to-end regression prevention.

## Why a skill?

You could do all of this manually. Read the code, form a hypothesis, try an optimization, eyeball the improvement. But the skill adds three things:

1. **Discipline.** It's too easy to skip the baseline, or to combine three changes before measuring, or to ship without checking edge cases. The skill's phased structure with explicit checkpoints makes it hard to cut corners.

2. **Breadth.** When you optimize manually, you tend to go with your first idea. The skill forces you to propose multiple strategies and compare them empirically. The winning strategy is often not the one you would have tried first.

3. **Documentation.** Every optimization run produces a benchmark table, a strategy comparison, and a final report. Six months later, when someone asks "why is this paginated?" or "what was the inbox dashboard response time before?", the answer is in the PR.

The skill is open source and available for any Claude Code project. Point it at your slowest endpoint and let it work.

---

*Built by [Marco Sanvido](https://github.com/msanvido). Inspired by [Karpathy's autoresearch](https://github.com/karpathy/autoresearch).*
