# Autoresearch

An autonomous experiment loop skill for Claude. Try an idea, measure it, keep what works, discard what doesn't, repeat forever.

Inspired by [Karpathy's autoresearch](https://github.com/davebcn87/pi-autoresearch) pattern: constraint + metric + autonomous iteration = compounding gains.

## What It Does

Autoresearch turns Claude into an autonomous optimization agent. Give it a measurable goal and a verification command, and it will loop indefinitely — making one atomic change at a time, measuring the result, keeping improvements, and reverting failures. No human intervention needed until you're ready to stop it.

It works on anything with a number you can measure: test speed, bundle size, code coverage, Lighthouse scores, API latency, build times, LLM training metrics, and more.

## How It Works

```
LOOP (forever or N times):
  1. Review state + git log + results log + session doc
  2. Pick next change (untried > near-misses > radical)
  3. Make ONE atomic change
  4. Git commit (before verification — enables clean rollback)
  5. Run verification command, parse metric
  6. Improved -> keep. Worse -> revert. Crash -> fix or revert.
  7. Log result to autoresearch-results.tsv
  8. Update autoresearch.md session document
  9. Repeat. NEVER STOP.
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
├── SKILL.md                                     # Main skill instructions
├── autoresearch.skill                           # Installable package
└── references/
    ├── autonomous-loop-protocol.md              # Detailed 9-phase loop protocol
    ├── core-principles.md                       # 7 universal principles
    └── results-logging.md                       # TSV format spec + reporting templates
```

## Key Design Decisions

- **One change per iteration.** Atomic changes mean you always know exactly what helped or hurt.
- **Commit before verify.** If verification hangs, `git revert HEAD --no-edit` cleanly restores state.
- **Stage only in-scope files.** Never `git add -A` — avoids accidentally committing secrets or unrelated files.
- **Mechanical verification only.** No subjective "looks good." The metric decides.
- **Simplicity wins.** Equal metric + less code = keep. Tiny gain + ugly complexity = discard.

## Credits

Based on the [pi-autoresearch](https://github.com/davebcn87/pi-autoresearch) extension by [@davebcn87](https://github.com/davebcn87), adapted for Claude.

## License

MIT
