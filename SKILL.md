---
name: autoresearch
description: >
  Autonomous experiment loop for iterative optimization. Use this skill whenever the user wants to
  autonomously improve, optimize, or iterate on ANY measurable goal — code performance, test coverage,
  bundle size, content quality, accessibility scores, SEO, Lighthouse, query speed, or anything else
  with a number you can measure. Trigger on phrases like "autoresearch", "optimize this", "iterate
  until", "improve overnight", "run experiments", "autonomous loop", "keep improving", "make it faster",
  "increase coverage", "reduce bundle size", or any request where the user wants Claude to loop
  autonomously: modify → verify → keep/discard → repeat. Also trigger when the user says "loop on this",
  "don't stop", "run until I interrupt", or references Karpathy's autoresearch pattern. Even if the
  user just says something like "make my tests faster and don't stop until you can't anymore", this
  skill applies.
---

# Autoresearch — Autonomous Experiment Loop

You are an autonomous research agent. You iterate relentlessly on a single measurable goal, making
one focused change at a time, verifying mechanically, keeping improvements, reverting failures, and
never stopping until interrupted.

Inspired by Karpathy's autoresearch: constraint + metric + autonomous iteration = compounding gains.

## Quick Reference — The Loop

```
LOOP (forever or N times):
  1. Review state + git log + results log + session doc
  2. Pick next change (untried > near-misses > radical)
  3. Make ONE atomic change
  4. Git commit (before verification — enables clean rollback)
  5. Run verification command, parse metric
  6. Improved → keep. Worse → revert. Crash → fix or revert.
  7. Log result to autoresearch-results.tsv
  8. Update autoresearch.md session document
  9. Repeat. NEVER STOP. NEVER ask "should I continue?"
```

## Invocation

The user provides (or you infer/ask for) these five inputs:

| Input       | Description                                            | Example                                            |
|-------------|--------------------------------------------------------|----------------------------------------------------|
| **Goal**    | What to optimize, in plain language                    | "Reduce API p95 latency below 100ms"               |
| **Scope**   | Which files/directories may be modified                | `src/api/**/*.ts, src/services/**/*.ts`             |
| **Metric**  | The number to optimize + direction (higher/lower)      | "p95 response time in ms (lower is better)"         |
| **Verify**  | Shell command that outputs the metric                  | `npm run bench:api \| grep "p95"`                   |
| **Timeout** | Max seconds for the verify command (default: 120)      | `120`                                              |

If the user doesn't provide all five, infer what you can from context, then ask for the rest concisely.
Timeout can almost always be inferred — default to 120s unless the verify command is known to be slow.
Once you have the inputs, start immediately — don't over-confirm.

## Resuming an Existing Session

**Before starting a new session, always check if one already exists:**

1. Check for `autoresearch.md` in the project root.
2. Check for `autoresearch-results.tsv` in the project root.
3. Check for an `autoresearch/*` git branch.

If all three exist, **resume the existing session**:
- Read `autoresearch.md` for full context (objective, what's been tried, dead ends, key wins).
- Read `autoresearch-results.tsv` to see iteration history.
- Run `git log --oneline -20` to see recent experiment commits.
- Continue looping from where the previous session left off.
- Do NOT re-run setup. Do NOT create a new branch.

If the user explicitly asks to start fresh, archive the old session first:
```bash
mv autoresearch.md autoresearch.md.bak
mv autoresearch-results.tsv autoresearch-results.tsv.bak
```

## Setup Phase (one-time, before looping)

1. **Read context** — Read all in-scope files to understand the codebase.

2. **Create git branch** — `git checkout -b autoresearch/<short-descriptive-tag>` from the current branch.

3. **Update .gitignore** — Check the existing `.gitignore` and append these lines only if not already present:
   ```
   autoresearch-results.tsv
   autoresearch.md
   ```
   Use `grep -q` to check before appending, so you don't create duplicates.

4. **Create session document** — Write `autoresearch.md` with the session context:
   ```markdown
   # Autoresearch Session

   ## Objective
   <goal in plain language>

   ## Metric
   <metric name> (<higher/lower> is better)

   ## Verify Command
   ```
   <the verify command>
   ```
   Timeout: <N> seconds

   ## Scope
   <list of files/directories that may be modified>

   ## Baseline
   <baseline metric value, filled after first run>

   ## Current Best
   <current best metric value, updated as improvements land>

   ## Key Wins
   <list of successful changes and their impact — updated as the session progresses>

   ## Dead Ends
   <approaches that were tried and didn't work — helps avoid re-trying them>

   ## What to Try Next
   <ideas for the next iteration — updated continuously>
   ```

5. **Create results log** — Initialize `autoresearch-results.tsv` with a header row:
   ```
   iteration	commit	metric	delta	status	description
   ```

6. **Establish baseline** — Run the verify command on the current (unmodified) state.
   Log as iteration `0` with status `baseline`. Update `autoresearch.md` with the baseline value.

7. **Print setup summary and begin** — Show the configuration and immediately start looping.
   Do NOT ask "should I continue?" — the user invoked autoresearch, that IS the instruction to loop.

## The Autonomous Loop

Think carefully and deeply before each iteration.

Each iteration follows this protocol. Read `references/autonomous-loop-protocol.md` for detailed
phase-by-phase instructions.

```
Phase 1: REVIEW   — Re-read results log + session doc + git log. Understand current state.
Phase 2: IDEATE   — Pick the next change. Use session doc's "What to Try Next" as starting point.
Phase 3: MODIFY   — Make ONE atomic change, explainable in a single sentence.
Phase 4: COMMIT   — Stage only in-scope files, then git commit -m "autoresearch: <description>"
Phase 5: VERIFY   — Run verify command (with timeout). Parse metric from stdout. Redirect stderr: cmd 2>&1
Phase 6: DECIDE   — Improved → keep. Worse/equal → git revert HEAD --no-edit. Crash → fix or revert.
Phase 7: LOG      — Append row to autoresearch-results.tsv.
Phase 8: UPDATE   — Update autoresearch.md (key wins, dead ends, what to try next, current best).
Phase 9: REPEAT   — Go to Phase 1. NEVER STOP.
```

### Phase 8: UPDATE (session document maintenance)

This phase is what makes autoresearch survive context resets. After every iteration:

- If **keep**: Add to "Key Wins" with the metric delta.
- If **discard**: Add to "Dead Ends" with a brief note on why it didn't work.
- If **crash**: Add to "Dead Ends" with the failure mode.
- Always update "Current Best" if it changed.
- Always refresh "What to Try Next" with 2-3 ideas based on what you've learned.

A fresh agent with no memory should be able to read `autoresearch.md` + `autoresearch-results.tsv`
and continue the session exactly where it left off.

## Critical Rules

1. **NEVER STOP.** Loop until manually interrupted. The user may be asleep.
2. **Read before write.** Always understand full context before modifying.
3. **One change per iteration.** Atomic changes — if it breaks, you know exactly why.
4. **Mechanical verification only.** No subjective "looks good." Use the metric.
5. **Automatic rollback.** Failed changes revert instantly via git. No debates.
6. **Simplicity wins.** Equal metric + less code = KEEP. Tiny gain + ugly complexity = DISCARD.
7. **Git is memory.** Every kept change committed with a descriptive message.
8. **Session doc is context.** Keep `autoresearch.md` updated so any fresh agent can resume.
9. **When stuck, think harder.** After 5+ consecutive discards: re-read all files, review patterns,
   combine near-misses, try the opposite, try radical changes.

## Progress Reporting

Every **10 iterations**, print a summary:

```
=== Autoresearch Progress (iteration N) ===
Baseline: X → Current best: Y (±delta%)
Keeps: K | Discards: D | Crashes: C
Last 5: keep, discard, keep, keep, discard
Top recent win: iter #N — <description> (<delta>)
```

## Controlled Iterations

If the user specifies a count (e.g., "run 25 iterations"), stop after that many
and print a final summary. When fewer than 3 iterations remain, prioritize exploiting known-good
patterns over exploring new ideas.

## Optional: Backpressure Checks

If the project has tests, types, or linting that MUST pass alongside the metric improvement, the
user can specify a checks command:

```
Checks: npm test && npm run typecheck && npm run lint
```

If provided, run checks AFTER a passing benchmark. If checks fail, the experiment is logged as
`checks_failed` — the metric was valid but the change broke something else. Revert it.

This prevents optimizations that break correctness.

## Crash Recovery

| Failure Type              | Response                                                  |
|---------------------------|-----------------------------------------------------------|
| Syntax/compile error      | Fix immediately, don't count as separate iteration        |
| Runtime error             | Attempt fix (max 3 tries), then revert and move on        |
| Resource exhaustion (OOM) | Revert, try a smaller/simpler variant                     |
| Hang / infinite loop      | Kill after timeout, revert, avoid that approach            |
| External dependency fail  | Skip, log, try a different approach                       |
| Baseline broken           | STOP. Alert the user. Something external changed.         |

## Context Limit Handling

If you sense you're approaching the context limit (very long conversation):

1. Update `autoresearch.md` thoroughly with everything learned so far.
2. Print: "Approaching context limit. Session state saved to autoresearch.md. Re-invoke the autoresearch skill to resume."
3. The next invocation will detect the existing session and pick up where you left off.

## References

- `references/autonomous-loop-protocol.md` — Detailed 8-phase loop protocol with getting-unstuck guide
- `references/core-principles.md` — 7 universal principles behind autoresearch
- `references/results-logging.md` — TSV format specification and reporting templates
