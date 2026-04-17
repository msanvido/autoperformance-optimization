# autoperformance-optimization

An [agent skill](https://github.com/vercel-labs/skills) for systematic, data-driven performance optimization. Works with any agent in the [open skills ecosystem](https://github.com/vercel-labs/skills#available-agents) -- Claude Code, Cursor, Codex, OpenCode, and [many more](https://github.com/vercel-labs/skills#available-agents). Inspired by Karpathy's [autoresearch](https://github.com/karpathy/autoresearch), but applied to performance engineering.

Give your coding agent a slow endpoint or page, and it will benchmark it, propose optimization strategies, implement them one at a time, measure each against the baseline, iterate on the winners, and ship the best result -- all with your approval at every step.

## How it works

The skill follows a 5-phase process:

1. **Define & Baseline** -- Identify the slow path, build automated benchmarks
2. **Identify Solutions** -- Propose 3-5 optimization strategies with impact/effort/risk estimates
3. **Implement & Compare** -- Build each strategy in isolation, measure against baseline
4. **Iterate Winners** -- Take the best 2-3 strategies, refine each through 3 iterations
5. **Validate & Ship** -- Final benchmark, verify data integrity, write E2E perf tests

At each phase boundary, the skill pauses and asks for your confirmation before proceeding.

## Installation

### Option 1: `npx skills` (recommended)

Uses the [open agent skills CLI](https://github.com/vercel-labs/skills), which auto-detects your agent(s) and installs to the correct location.

```bash
# Install into the current project (auto-detects installed agents)
npx skills add msanvido/autoperformance-optimization

# Or install globally for all projects
npx skills add msanvido/autoperformance-optimization -g

# Target a specific agent explicitly
npx skills add msanvido/autoperformance-optimization -a claude-code
npx skills add msanvido/autoperformance-optimization -a cursor
```

See the [agents list](https://github.com/vercel-labs/skills#available-agents) for all supported agents.

### Option 2: Manual install

The canonical skill lives at [`skills/autoperformance-optimization/SKILL.md`](skills/autoperformance-optimization/SKILL.md). Drop that file (or symlink the folder) into whichever skills directory your agent reads -- e.g. `.claude/skills/autoperformance-optimization/` for Claude Code, or the equivalent path for your agent.

```bash
# Example for Claude Code
mkdir -p .claude/skills/autoperformance-optimization
curl -o .claude/skills/autoperformance-optimization/SKILL.md \
  https://raw.githubusercontent.com/msanvido/autoperformance-optimization/main/skills/autoperformance-optimization/SKILL.md
```

## Usage

Once installed, the skill activates automatically when you ask your agent to optimize performance. Trigger phrases include:

- "optimize performance of `/api/dashboard`"
- "this endpoint is too slow, speed it up"
- "make the activity digest page faster"
- "benchmark and improve the inbox API response time"
- "reduce load time for the settings page"

Your agent will then walk you through the 5-phase process, asking for your input at each decision point.

## Example results

This skill has been used to deliver significant performance improvements on production systems -- and, just as importantly, to prevent shipping regressions disguised as optimizations:

| Target | Before | After | Improvement |
|--------|--------|-------|-------------|
| Inbox dashboard API | ~6s (warm) / ~11s (cold) | ~500ms (warm) / ~4s (cold) | 11x / 2.6x faster |
| Activity digest page | ~20s (full load) | ~2.5s (first page) | 8x faster |
| [nanochat](https://github.com/karpathy/nanochat) training loop (M3 Max MPS) | 607 ms/step | 607 ms/step | **No change -- all 3 proposed strategies regressed under bit-exact constraints; shipping nothing was the correct answer** |

See [BLOG.md](BLOG.md) for detailed write-ups of these optimizations.

## Philosophy

- **Never optimize without measuring first.** You cannot know if you made things better or worse without a baseline.
- **Avoid premature optimization.** Only optimize code paths that have stabilized in the codebase. Optimizing a moving target wastes effort and creates merge conflicts.
- **Re-benchmark after handling corner cases.** The common path may be fast, but fixing edge cases can introduce unexpected slowdowns. Always re-run the full benchmark suite after every change.
- **Correctness over speed.** Performance improvements must not sacrifice data integrity. Always verify that counts match, data types are preserved, and filters still work.

## License

MIT
