# Autoresearch

An autonomous experiment loop skill for Claude. Try an idea, measure it, keep what works, discard what doesn't, repeat forever.

Inspired by [Karpathy's autoresearch](https://github.com/davebcn87/pi-autoresearch) pattern: constraint + metric + autonomous iteration = compounding gains.

## What It Does

Autoresearch turns Claude into an autonomous optimization agent. Give it a measurable goal and a verification command, and it will loop indefinitely — making one atomic change at a time, measuring the result, keeping improvements, and reverting failures. No human intervention needed until you're ready to stop it.

It works on anything with a number you can measure: test speed, bundle size, code coverage, Lighthouse scores, API latency, build times, LLM training metrics, and more.

## Multi-Agent Architecture

Autoresearch uses an **orchestrator + subagent** pattern to stay within context limits during long-running loops.

```
┌──────────────────────────────────────────────────┐
│  ORCHESTRATOR (main agent — stays lean)          │
│  Reads only: autoresearch.md + results TSV       │
│  Does: review → ideate → spawn subagent          │
│        ← receive compact result                  │
│        decide → log → update docs → repeat       │
│  ~200 tokens per iteration                       │
├──────────────────────────────────────────────────┤
│  EXPERIMENT SUBAGENT (disposable, one per iter)  │
│  Reads: source files, runs commands              │
│  Does: modify → commit → verify → return result  │
│  Context discarded after returning               │
└──────────────────────────────────────────────────┘
```

Without subagents, the context fills up after ~15-20 iterations as source files and command output accumulate. With this architecture, the orchestrator can run **100+ iterations** because it only sees compact results from each subagent.

The skill falls back to single-agent mode automatically if subagents aren't available.

## How It Works

```
ORCHESTRATOR LOOP (forever or N times):
  1. REVIEW   — Read session doc + results log + git log
  2. IDEATE   — Pick next change from wins, dead ends, and ideas
  3. DELEGATE — Spawn subagent with change description
     └─ SUBAGENT: read files → modify → commit → verify → return result
  4. DECIDE   — Parse result: keep / revert / fix
  5. LOG      — Append row to results TSV
  6. UPDATE   — Update session doc with outcome
  7. REPEAT   — Never stop. Never ask "should I continue?"
```

## Installation

Copy the skill folder into your Claude skills directory, or install the `.skill` package directly.

## Usage

Tell Claude something like:

- "Make my tests faster and don't stop until you can't anymore"
- "Optimize the bundle size of my app"
- "Run autoresearch on test coverage"
- "Iterate on API latency until I interrupt you"

Claude will ask for (or infer) five inputs:

| Input       | Description                                       | Example                              |
|-------------|---------------------------------------------------|--------------------------------------|
| **Goal**    | What to optimize, in plain language                | "Reduce API p95 latency below 100ms" |
| **Scope**   | Which files/directories may be modified            | `src/api/**/*.ts`                    |
| **Metric**  | The number to optimize + direction                 | "p95 in ms (lower is better)"        |
| **Verify**  | Shell command that outputs the metric              | `npm run bench:api \| grep "p95"`    |
| **Timeout** | Max seconds for the verify command (default: 120)  | `120`                                |

Then it creates a git branch, establishes a baseline, and starts looping.

## Session Persistence

Autoresearch survives context resets. Every iteration updates two files:

- **autoresearch.md** — Living session document with objective, key wins, dead ends, and next ideas. A fresh agent reading this file can resume exactly where the last one left off.
- **autoresearch-results.tsv** — Append-only log of every experiment (iteration, commit hash, metric, delta, status, description).

## File Structure

```
autoresearch/
├── SKILL.md                                     # Main skill — orchestrator + subagent architecture
├── autoresearch.skill                           # Installable package
└── references/
    ├── autonomous-loop-protocol.md              # Detailed protocol with subagent prompt templates
    ├── core-principles.md                       # 7 universal principles
    └── results-logging.md                       # TSV format spec + reporting templates
```

## Key Design Decisions

- **Multi-agent by default.** Subagents handle file reads and command execution so the orchestrator stays lean. Falls back to single-agent mode when subagents aren't available.
- **One change per iteration.** Atomic changes mean you always know exactly what helped or hurt.
- **Commit before verify.** If verification hangs, `git revert HEAD --no-edit` cleanly restores state.
- **Stage only in-scope files.** Never `git add -A` — avoids accidentally committing secrets or unrelated files.
- **Mechanical verification only.** No subjective "looks good." The metric decides.
- **Simplicity wins.** Equal metric + less code = keep. Tiny gain + ugly complexity = discard.
- **Adaptive model selection.** Uses sonnet for fast iterations, escalates to opus when stuck.

## Credits

Based on the [pi-autoresearch](https://github.com/davebcn87/pi-autoresearch) extension by [@davebcn87](https://github.com/davebcn87), adapted for Claude with multi-agent architecture.

## License

MIT
