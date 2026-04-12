# autoperformance-optimization

A [Claude Code](https://claude.ai/code) skill for systematic, data-driven performance optimization. Inspired by Karpathy's [autoresearch](https://github.com/karpathy/autoresearch), but applied to performance engineering.

Give Claude Code a slow endpoint or page, and it will benchmark it, propose optimization strategies, implement them one at a time, measure each against the baseline, iterate on the winners, and ship the best result -- all with your approval at every step.

## How it works

The skill follows a 5-phase process:

1. **Define & Baseline** -- Identify the slow path, build automated benchmarks
2. **Identify Solutions** -- Propose 3-5 optimization strategies with impact/effort/risk estimates
3. **Implement & Compare** -- Build each strategy in isolation, measure against baseline
4. **Iterate Winners** -- Take the best 2-3 strategies, refine each through 3 iterations
5. **Validate & Ship** -- Final benchmark, verify data integrity, write E2E perf tests

At each phase boundary, the skill pauses and asks for your confirmation before proceeding.

## Installation

### Option 1: Add as a git submodule (recommended)

```bash
# From your project root
git submodule add https://github.com/msanvido/autoperformance-optimization.git .claude/skills/autoperformance-optimization
```

This keeps the skill pinned to a specific commit and makes updates explicit via `git submodule update --remote`.

### Option 2: Copy the skill file directly

```bash
# From your project root
mkdir -p .claude/skills
curl -o .claude/skills/autoperformance-optimization.md \
  https://raw.githubusercontent.com/msanvido/autoperformance-optimization/main/.claude/skills/SKILL.md
```

### Option 3: Clone and symlink

```bash
# Clone the repo somewhere on your machine
git clone https://github.com/msanvido/autoperformance-optimization.git ~/tools/autoperformance-optimization

# Symlink the skill into your project
mkdir -p .claude/skills
ln -s ~/tools/autoperformance-optimization/.claude/skills/SKILL.md .claude/skills/autoperformance-optimization.md
```

## Usage

Once installed, the skill activates automatically when you ask Claude Code to optimize performance. Trigger phrases include:

- "optimize performance of `/api/dashboard`"
- "this endpoint is too slow, speed it up"
- "make the activity digest page faster"
- "benchmark and improve the inbox API response time"
- "reduce load time for the settings page"

Claude Code will then walk you through the 5-phase process, asking for your input at each decision point.

## Example results

This skill has been used to deliver significant performance improvements on production systems:

| Target | Before | After | Improvement |
|--------|--------|-------|-------------|
| Inbox dashboard API | ~6s (warm) / ~11s (cold) | ~500ms (warm) / ~4s (cold) | 11x / 2.6x faster |
| Activity digest page | ~20s (full load) | ~2.5s (first page) | 8x faster |

See [BLOG.md](BLOG.md) for detailed write-ups of these optimizations.

## Philosophy

- **Never optimize without measuring first.** You cannot know if you made things better or worse without a baseline.
- **Avoid premature optimization.** Only optimize code paths that have stabilized in the codebase. Optimizing a moving target wastes effort and creates merge conflicts.
- **Re-benchmark after handling corner cases.** The common path may be fast, but fixing edge cases can introduce unexpected slowdowns. Always re-run the full benchmark suite after every change.
- **Correctness over speed.** Performance improvements must not sacrifice data integrity. Always verify that counts match, data types are preserved, and filters still work.

## License

MIT
