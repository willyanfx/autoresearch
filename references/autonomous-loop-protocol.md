# Autonomous Loop Protocol — Detailed Phase Instructions

## Phase 1: REVIEW

Before every iteration, refresh your understanding:

1. Re-read `autoresearch-results.tsv` to see what's been tried and what worked.
2. Re-read `autoresearch.md` — especially "Dead Ends" (don't repeat) and "What to Try Next."
3. Run `git log --oneline -10` to see recent commits and reverts.
4. Re-read any files that are relevant to your next idea.
5. Identify patterns: What categories of changes tend to succeed? What keeps failing?

**Do not skip this phase.** Context drift is the #1 cause of wasted iterations.

## Phase 2: IDEATE

Pick the next change using this priority order:

1. **Low-hanging fruit** — Simple changes with high expected impact that haven't been tried.
2. **Variations on successes** — If a category of change worked, try more from that category.
3. **Near-misses** — Changes that almost improved the metric. Can you refine them?
4. **Combinations** — Can you combine two small wins into a bigger one?
5. **Radical ideas** — If conservative changes are exhausted, try something bold.
6. **Opposite approach** — If you've been stuck, try the exact opposite of your recent strategy.

Write a one-sentence description of the change before making it. This becomes the commit message
and the log entry.

**Check "Dead Ends" in autoresearch.md before proceeding.** If this approach was already tried and
failed, pick something else. Don't waste iterations on known failures.

## Phase 3: MODIFY

Make ONE atomic change:

- The change should be explainable in a single sentence.
- Do not bundle multiple unrelated changes. If change A helps and change B hurts, bundling them
  together means you lose both.
- If a change requires modifying multiple files, that's fine — "atomic" means "one logical idea."
- Prefer the simplest implementation. You can always iterate to a more sophisticated version later.

## Phase 4: COMMIT

Commit BEFORE running verification:

```bash
git add <in-scope files that changed>
git commit -m "autoresearch: <one-sentence description>"
```

Stage only the files within the declared scope — never use `git add -A` or `git add .` as this
risks staging secrets (.env, credentials), large binaries, or unrelated files. Use specific paths
or glob patterns that match the scope. The session doc and results TSV should already be in
`.gitignore` and won't be staged.

Why commit first? Because if verification fails or hangs, you need a clean state to revert to.
`git revert HEAD --no-edit` cleanly undoes exactly one commit.

## Phase 5: VERIFY

Run the user's verify command and capture output:

```bash
<verify_command> 2>&1
```

Parse the metric from the output. The metric should be a number. Common patterns:
- Grep for a specific line: `npm test -- --coverage | grep "All files"`
- Look for a labeled value: `SCORE: 87.3`
- Read the last line of output
- Parse JSON output with jq

If the verify command exits with a non-zero code, treat it as a crash (see Phase 6).

**Timeout:** Kill the verify command if it exceeds the configured timeout (default 120s). If the
user provided a custom timeout in the session config, use that value. For commands known to be slow
(e.g., full test suites, model training), a longer timeout is appropriate.

### If backpressure checks are configured

After a passing verification (metric parsed successfully), run the checks command:

```bash
<checks_command> 2>&1
```

If checks fail, treat the result as `checks_failed` — revert the change even though the metric
improved. The metric improvement doesn't count if it broke correctness.

## Phase 6: DECIDE

Three possible outcomes:

### IMPROVED (metric better than current best)
- Status: `keep`
- Action: Keep the commit. Update the current best metric.
- The commit stays on the branch.

### WORSE OR EQUAL (metric same or worse than current best)
- Status: `discard`
- Action: Revert the commit immediately.
  ```bash
  git revert HEAD --no-edit
  ```

### CRASHED (verify command failed, error, or unparseable output)
- Status: `crash`
- Action: Attempt to fix the issue (max 3 attempts). If fixed, re-run verification.
  If not fixable after 3 attempts, revert and move on.
  ```bash
  git revert HEAD --no-edit
  ```

### CHECKS FAILED (metric improved but correctness checks failed)
- Status: `checks_failed`
- Action: Revert. The metric gain isn't worth the correctness regression.
  ```bash
  git revert HEAD --no-edit
  ```

### Simplicity criterion
If the metric improved by a trivial amount (< 1% relative improvement) but the change adds
significant complexity (many new lines, new dependencies, convoluted logic), DISCARD it.
Small gains that make the codebase harder to maintain are not worth keeping.

## Phase 7: LOG

Append a row to `autoresearch-results.tsv`:

```
<iteration>	<commit_hash_or_dash>	<metric_value>	<delta>	<status>	<description>
```

- `iteration`: Sequential number starting from 0 (baseline).
- `commit`: The short commit hash if kept, `-` if discarded/crashed.
- `metric`: The measured value (or `0.0` / `ERR` if crashed).
- `delta`: Change from baseline (e.g., `+2.3` or `-0.5`). Use `0.0` for baseline.
- `status`: One of `baseline`, `keep`, `discard`, `crash`, `checks_failed`.
- `description`: One-sentence description of what was tried.

## Phase 8: UPDATE

Update `autoresearch.md` to maintain session continuity:

- **If keep:** Add to "Key Wins" section: `- iter #N: <description> (<delta>)`
- **If discard:** Add to "Dead Ends" section: `- <description> — <why it didn't work>`
- **If crash:** Add to "Dead Ends" section: `- <description> — CRASHED: <failure mode>`
- **If checks_failed:** Add to "Dead Ends" section: `- <description> — metric improved but broke checks`
- **Always** update "Current Best" if it changed.
- **Always** refresh "What to Try Next" with 2-3 ideas informed by what you just learned.

This is the most important phase for long-running sessions. A fresh agent reading `autoresearch.md`
should understand exactly what has been tried, what worked, what failed, and what to try next.

## Phase 9: REPEAT

Return to Phase 1. Do not pause. Do not ask for confirmation. Do not summarize unless it's a
10-iteration checkpoint. The loop is autonomous.

---

## Getting Unstuck

### After 5+ consecutive discards:

1. Stop and re-read ALL in-scope files from scratch.
2. Review the full results log for patterns.
3. Re-read "Key Wins" — can you extend any of those successful approaches?
4. List what has NOT been tried yet.
5. Consider combining two near-miss changes.
6. Try a fundamentally different approach (different algorithm, data structure, etc.).
7. If the metric seems to have plateaued, note it but keep trying. Many plateaus have a cliff
   on the other side if you find the right angle.

### After 3+ consecutive crashes:

1. The verify command or environment may be broken.
2. Re-run the verify command on the current (clean) state to confirm the baseline still works.
3. If the baseline is broken, STOP and alert the user.
4. If the baseline works, the crashes are from your changes — try simpler, more conservative edits.

### When approaching diminishing returns:

If the last 20 iterations have all been discards or trivial improvements (< 0.1% each):

1. Print a status update about diminishing returns.
2. Consider whether the goal is achievable or if you've hit a natural limit.
3. Try 5 more radical/unconventional approaches before suggesting to the user that the metric
   may have plateaued.
4. If radical approaches also fail, print a summary and note that further improvement may require
   architectural changes outside the current scope.
5. Keep looping unless explicitly told to stop — the user chose "never stop" for a reason.
