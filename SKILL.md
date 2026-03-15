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

## Architecture — Orchestrator + Experiment Subagents

This skill uses a **multi-agent architecture** to conserve context. The main conversation acts as
a lightweight orchestrator that never reads source files or command output directly. Instead, each
experiment runs inside a disposable subagent whose context is discarded after it reports back.

```
┌─────────────────────────────────────────────────────┐
│  ORCHESTRATOR (main agent — stays lean)             │
│                                                     │
│  Reads: autoresearch.md, autoresearch-results.tsv   │
│  Does:  REVIEW → IDEATE → spawn subagent            │
│         ← receive result                            │
│         DECIDE → LOG → UPDATE → repeat              │
│                                                     │
│  Never reads: source files, command output           │
│  Context cost per iteration: ~200 tokens            │
├─────────────────────────────────────────────────────┤
│  EXPERIMENT SUBAGENT (one per iteration — disposable)│
│                                                     │
│  Reads: in-scope source files                       │
│  Does:  MODIFY → COMMIT → VERIFY                   │
│  Returns: { metric, status, commit, description }   │
│                                                     │
│  Context is discarded after returning result        │
│  Context cost to orchestrator: 0                    │
└─────────────────────────────────────────────────────┘
```

**Why this matters:** Without subagents, every iteration loads source files and verification output
into the main context. After 15-20 iterations, the context is full and the session dies. With
subagents, the orchestrator only accumulates ~200 tokens per iteration (the compact result), so it
can run 100+ iterations before hitting limits.

## Quick Reference — The Loop

```
ORCHESTRATOR LOOP (forever or N times):
  1. REVIEW   — Read autoresearch.md + results TSV + git log (compact state only)
  2. IDEATE   — Pick next change based on wins, dead ends, and ideas
  3. DELEGATE — Spawn experiment subagent with change description
     └─ SUBAGENT: read files → modify → commit → verify → return result
  4. DECIDE   — Parse subagent result: keep / revert / fix
  5. LOG      — Append row to autoresearch-results.tsv
  6. UPDATE   — Update autoresearch.md (key wins, dead ends, what to try next)
  7. REPEAT   — Go to step 1. NEVER STOP. NEVER ask "should I continue?"
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

## The Orchestrator Loop

Think carefully and deeply before each iteration. Read `references/autonomous-loop-protocol.md` for
detailed phase-by-phase instructions including the subagent prompt template.

### Step 1: REVIEW (orchestrator)

Re-read `autoresearch.md` and `autoresearch-results.tsv`. Run `git log --oneline -10`.
These are small files — they're the only things the orchestrator loads each iteration.
Identify patterns in what's working, what's failing, and what hasn't been tried.

### Step 2: IDEATE (orchestrator)

Pick the next change. Use the "What to Try Next" section from the session doc as a starting
point. Check "Dead Ends" to avoid repeating failures. Write a one-sentence description of the
change you want to try.

### Step 3: DELEGATE (spawn experiment subagent)

Spawn a subagent with the Agent tool using this prompt structure:

```
You are an autoresearch experiment agent. Your job is to make ONE atomic change to improve
a metric, commit it, and run verification.

GOAL: <goal>
SCOPE: <files/directories in scope>
METRIC: <metric name> (<direction> is better)
VERIFY COMMAND: <command>
TIMEOUT: <N> seconds
CURRENT BEST: <current best metric value>
BASELINE: <baseline metric value>

CHANGE TO TRY: <one-sentence description of the change from IDEATE step>

INSTRUCTIONS:
1. Read the relevant in-scope files to understand the current code.
2. Make the described change — ONE atomic modification.
3. Stage only in-scope files and commit: git commit -m "autoresearch: <description>"
4. Run the verify command with timeout: timeout <N>s <verify_command> 2>&1
5. Parse the metric from the output.

RESPOND WITH EXACTLY THIS FORMAT (nothing else):
METRIC: <number or ERR>
STATUS: <keep|discard|crash|checks_failed>
COMMIT: <short hash or REVERTED>
DESCRIPTION: <one-sentence summary of what you did>
NOTES: <brief context for the orchestrator — why it worked/failed, what to try next>

DECISION RULES:
- If the metric improved vs current best → STATUS: keep (leave the commit)
- If the metric is worse or equal → STATUS: discard (run: git revert HEAD --no-edit)
- If the verify command crashed → STATUS: crash (attempt fix up to 3 times, then revert)
- If metric improved but < 1% gain with significant complexity → STATUS: discard (revert)
```

Use `subagent_type: "general-purpose"` and set `model: "sonnet"` for the subagent to keep
iteration costs low. Use `model: "opus"` only if you're stuck (5+ consecutive discards).

### Step 4: DECIDE (orchestrator)

Parse the subagent's structured response. The subagent already handled the keep/revert decision
and git operations. You just need to record the result.

### Step 5: LOG (orchestrator)

Append a row to `autoresearch-results.tsv`:
```
<iteration>	<commit_hash_or_dash>	<metric_value>	<delta>	<status>	<description>
```

### Step 6: UPDATE (orchestrator)

Update `autoresearch.md` — this is what makes the session survive context resets:

- If **keep**: Add to "Key Wins" with the metric delta.
- If **discard**: Add to "Dead Ends" with a brief note on why (from subagent NOTES).
- If **crash**: Add to "Dead Ends" with the failure mode.
- Always update "Current Best" if it changed.
- Always refresh "What to Try Next" with 2-3 ideas based on what you've learned.

### Step 7: REPEAT

Go to Step 1. Do not pause. Do not ask for confirmation. The loop is autonomous.

## Fallback: Single-Agent Mode

If the Agent tool is unavailable (no subagent support), fall back to running everything in
the main context. This uses more context per iteration but still works. Follow the same phases
but execute MODIFY, COMMIT, and VERIFY directly instead of delegating.

When running in single-agent mode, be extra diligent about context limit handling (see below).

## Critical Rules

1. **NEVER STOP.** Loop until manually interrupted. The user may be asleep.
2. **Delegate the heavy work.** Source files and command output belong in subagents, not the orchestrator.
3. **One change per iteration.** Atomic changes — if it breaks, you know exactly why.
4. **Mechanical verification only.** No subjective "looks good." Use the metric.
5. **Automatic rollback.** Failed changes revert instantly via git. No debates.
6. **Simplicity wins.** Equal metric + less code = KEEP. Tiny gain + ugly complexity = DISCARD.
7. **Git is memory.** Every kept change committed with a descriptive message.
8. **Session doc is context.** Keep `autoresearch.md` updated so any fresh agent can resume.
9. **When stuck, escalate.** After 5+ consecutive discards: switch subagent to opus model,
   re-read all files in the subagent, try radical/opposite approaches.

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

If provided, include the checks command in the subagent prompt. The subagent runs checks AFTER
a passing benchmark. If checks fail, the experiment is logged as `checks_failed`.

## Crash Recovery

| Failure Type              | Response                                                  |
|---------------------------|-----------------------------------------------------------|
| Syntax/compile error      | Subagent fixes immediately, doesn't count as separate iter|
| Runtime error             | Subagent attempts fix (max 3 tries), then reverts         |
| Resource exhaustion (OOM) | Subagent reverts, orchestrator notes to try simpler variant|
| Hang / infinite loop      | Subagent kills after timeout, reverts                     |
| External dependency fail  | Subagent skips, orchestrator tries different approach      |
| Baseline broken           | STOP. Alert the user. Something external changed.         |
| Subagent fails to respond | Log as crash, continue with next iteration                |

## Context Limit Handling

The multi-agent architecture makes context limits much less likely, but if you sense you're
approaching the limit (very long conversation):

1. Update `autoresearch.md` thoroughly with everything learned so far.
2. Print: "Approaching context limit. Session state saved to autoresearch.md. Re-invoke the autoresearch skill to resume."
3. The next invocation will detect the existing session and pick up where you left off.

## References

- `references/autonomous-loop-protocol.md` — Detailed phase-by-phase protocol with subagent prompts and getting-unstuck guide
- `references/core-principles.md` — 7 universal principles behind autoresearch
- `references/results-logging.md` — TSV format specification and reporting templates
