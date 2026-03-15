# Autonomous Loop Protocol — Detailed Phase Instructions

This document covers the multi-agent loop protocol. The orchestrator (main agent) stays lean by
delegating experiment execution to disposable subagents. Each subagent reads files, makes changes,
and runs verification — then returns a compact result and its context is discarded.

---

## Orchestrator Phases

These phases run in the main agent. They work with compact data only: the session document,
the results TSV, and git log output. The orchestrator should never read source files directly.

### Phase 1: REVIEW (orchestrator)

Before every iteration, refresh your understanding from the compact state files:

1. Re-read `autoresearch-results.tsv` — see what's been tried and what worked.
2. Re-read `autoresearch.md` — especially "Dead Ends" (don't repeat) and "What to Try Next."
3. Run `git log --oneline -10` to see recent commits and reverts.
4. Identify patterns: What categories of changes tend to succeed? What keeps failing?

**Do not read source files in the orchestrator.** That's the subagent's job. The orchestrator
reasons about strategy from the session doc and results log only.

### Phase 2: IDEATE (orchestrator)

Pick the next change using this priority order:

1. **Low-hanging fruit** — Simple changes with high expected impact that haven't been tried.
2. **Variations on successes** — If a category of change worked, try more from that category.
3. **Near-misses** — Changes that almost improved the metric. Can you refine them?
4. **Combinations** — Can you combine two small wins into a bigger one?
5. **Radical ideas** — If conservative changes are exhausted, try something bold.
6. **Opposite approach** — If you've been stuck, try the exact opposite of your recent strategy.

Write a one-sentence description of the change. This becomes the subagent's mission, the commit
message, and the log entry.

**Check "Dead Ends" in autoresearch.md before proceeding.** If this approach was already tried and
failed, pick something else. Don't waste iterations on known failures.

### Phase 3: DELEGATE (orchestrator → subagent)

Spawn an experiment subagent using the Agent tool. Use `model: "sonnet"` by default for fast,
cheap iterations. Switch to `model: "opus"` when stuck (5+ consecutive discards) for deeper
reasoning.

#### Subagent Prompt Template

```
You are an autoresearch experiment agent. Your job is to make ONE atomic change to improve
a metric, commit it, and run verification. Work autonomously — do not ask questions.

SESSION CONTEXT:
- Goal: <goal>
- Scope: <files/directories in scope>
- Metric: <metric name> (<direction> is better)
- Verify command: <command>
- Timeout: <N> seconds
- Current best: <current best metric value>
- Baseline: <baseline metric value>
- Checks command: <checks command or "none">

CHANGE TO TRY: <one-sentence description>

DEAD ENDS (do NOT retry these):
<paste the Dead Ends section from autoresearch.md>

INSTRUCTIONS:
1. Read the relevant in-scope files to understand current code.
2. Make the described change — ONE atomic modification, explainable in one sentence.
   Keep it simple. If equal metric + less code, that's a win.
3. Stage only in-scope files and commit:
   git add <specific files>
   git commit -m "autoresearch: <description>"
4. Run verification:
   timeout <N>s <verify_command> 2>&1
5. Parse the metric value from the output.
6. If checks command is configured and metric improved, also run:
   timeout <N>s <checks_command> 2>&1
7. Make the keep/revert decision:
   - Metric improved (and checks passed if configured) → keep the commit
   - Metric worse, equal, or crashed → git revert HEAD --no-edit
   - Metric improved < 1% but adds significant complexity → git revert HEAD --no-edit
   - Checks failed → git revert HEAD --no-edit (status: checks_failed)

RESPOND WITH EXACTLY THIS FORMAT (nothing else after NOTES):
METRIC: <number or ERR>
STATUS: <keep|discard|crash|checks_failed>
COMMIT: <7-char hash or REVERTED>
DESCRIPTION: <one-sentence summary of what you did>
NOTES: <2-3 sentences: why it worked/failed, observations about the code, what might work next>
```

#### Subagent Guidelines

- Set `description: "autoresearch iteration N"` on the Agent tool call.
- The subagent handles everything: reading files, editing, committing, running verification,
  and making the keep/revert decision. The orchestrator just records the result.
- If a subagent fails to return a properly formatted response, log the iteration as `crash`
  with COMMIT: `-` and continue.
- Parse the subagent's response by looking for the METRIC/STATUS/COMMIT/DESCRIPTION/NOTES lines.

### Phase 4: DECIDE (orchestrator)

Parse the subagent's structured result. The subagent already handled git operations (keep or
revert). The orchestrator just needs to:

1. Extract METRIC, STATUS, COMMIT, DESCRIPTION, NOTES from the response.
2. Calculate delta from baseline.
3. Update current best if STATUS is `keep`.

### Phase 5: LOG (orchestrator)

Append a row to `autoresearch-results.tsv`:

```
<iteration>	<commit_hash_or_dash>	<metric_value>	<delta>	<status>	<description>
```

- `iteration`: Sequential number starting from 0 (baseline).
- `commit`: The short commit hash if kept, `-` if discarded/crashed.
- `metric`: The measured value (or `ERR` if crashed).
- `delta`: Change from baseline (e.g., `+2.3` or `-0.5`). Use `0.0` for baseline.
- `status`: One of `baseline`, `keep`, `discard`, `crash`, `checks_failed`.
- `description`: One-sentence description of what was tried.

### Phase 6: UPDATE (orchestrator)

Update `autoresearch.md` to maintain session continuity. Use the subagent's NOTES to inform
your updates — they contain observations about the code that the orchestrator can't see directly.

- **If keep:** Add to "Key Wins" section: `- iter #N: <description> (<delta>)`
- **If discard:** Add to "Dead Ends" section: `- <description> — <reason from NOTES>`
- **If crash:** Add to "Dead Ends" section: `- <description> — CRASHED: <failure from NOTES>`
- **If checks_failed:** Add to "Dead Ends" section: `- <description> — metric improved but broke checks`
- **Always** update "Current Best" if it changed.
- **Always** refresh "What to Try Next" with 2-3 ideas informed by the subagent's NOTES.

This phase is critical. The NOTES field from each subagent carries forward observations that would
otherwise be lost when the subagent's context is discarded. Weave them into the session doc so
future subagents benefit from past observations.

### Phase 7: REPEAT

Return to Phase 1. Do not pause. Do not ask for confirmation. Do not summarize unless it's a
10-iteration checkpoint. The loop is autonomous.

---

## Getting Unstuck

### After 5+ consecutive discards:

1. Switch the subagent model to `opus` for deeper reasoning.
2. Include more context in the subagent prompt: paste the full "Key Wins" section so the
   subagent can build on what's worked before.
3. Ask the subagent to try a fundamentally different approach (different algorithm, data
   structure, etc.).
4. Consider combining two near-miss changes into one subagent prompt.
5. Try the exact opposite of your recent strategy.
6. If the metric seems to have plateaued, note it in the session doc but keep trying.
   Many plateaus have a cliff on the other side if you find the right angle.

### After 3+ consecutive crashes:

1. The verify command or environment may be broken.
2. Spawn a diagnostic subagent that just runs the verify command on the current (clean) state.
3. If the baseline is broken, STOP and alert the user.
4. If the baseline works, the crashes are from your changes — instruct the next subagent to
   try simpler, more conservative edits.

### When approaching diminishing returns:

If the last 20 iterations have all been discards or trivial improvements (< 0.1% each):

1. Print a status update about diminishing returns.
2. Consider whether the goal is achievable or if you've hit a natural limit.
3. Switch to opus model and try 5 more radical/unconventional approaches.
4. If radical approaches also fail, print a summary and note that further improvement may require
   architectural changes outside the current scope.
5. Keep looping unless explicitly told to stop — the user chose "never stop" for a reason.

---

## Subagent Cost Optimization

The multi-agent approach has a secondary benefit: you can tune the cost/quality tradeoff per
iteration. Here's a rough guide:

| Situation                       | Subagent Model | Reasoning                             |
|---------------------------------|----------------|---------------------------------------|
| Normal iteration                | sonnet         | Fast, cheap, good for simple changes  |
| Stuck (5+ discards)             | opus           | Deeper reasoning, more creative       |
| Near the goal (< 5% remaining) | opus           | Precision matters more than speed     |
| Crash recovery                  | sonnet         | Quick diagnostic, don't overthink     |
| Radical/architectural change    | opus           | Needs strong understanding of codebase|

---

## Single-Agent Fallback

If the Agent tool is unavailable, the orchestrator runs everything directly. In this mode:

- Execute Phases 1-7 inline (no delegation).
- Be extra vigilant about context limits — you're accumulating file contents and command output.
- Prefer reading only the specific files you plan to modify, not the entire scope.
- After each verify command, focus on parsing the metric and discard verbose output from memory.
- Consider approaching context limit after ~15-20 iterations and save state proactively.
