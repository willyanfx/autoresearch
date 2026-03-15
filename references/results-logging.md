# Results Logging Format

## TSV Structure

The results log is a tab-separated file named `autoresearch-results.tsv` in the project root.

### Header

```
iteration	commit	metric	delta	status	description
```

### Row Format

| Column        | Type    | Description                                              |
|---------------|---------|----------------------------------------------------------|
| `iteration`   | integer | Sequential number, starting from 0 (baseline)           |
| `commit`      | string  | Short git hash (7 chars) if kept, `-` if discarded/crash |
| `metric`      | float   | The measured value from the verify command                |
| `delta`       | float   | Change from baseline (signed, e.g. `+2.3` or `-0.5`)    |
| `status`      | string  | One of: `baseline`, `keep`, `discard`, `crash`, `checks_failed` |
| `description` | string  | One-sentence summary of what was tried                   |

### Example

```
iteration	commit	metric	delta	status	description
0	a1b2c3d	85.2	0.0	baseline	initial state — test coverage 85.2%
1	b2c3d4e	87.1	+1.9	keep	add tests for auth middleware edge cases
2	-	86.5	-0.6	discard	refactor test helpers (broke 2 tests)
3	-	0.0	0.0	crash	add integration tests (DB connection failed)
4	c3d4e5f	88.3	+3.1	keep	add tests for error handling in API routes
5	-	89.0	+3.8	checks_failed	inline test helpers (coverage up but typecheck broke)
6	d4e5f6g	89.0	+3.8	keep	add boundary value tests for validators
```

## Progress Summary Template

Print every 10 iterations:

```
=== Autoresearch Progress (iteration N) ===
Baseline: <baseline_metric> → Current best: <best_metric> (<delta_from_baseline>)
Keeps: <count> | Discards: <count> | Crashes: <count>
Last 5: <status>, <status>, <status>, <status>, <status>
Top wins:
  - iter <N>: <description> (<delta>)
  - iter <N>: <description> (<delta>)
  - iter <N>: <description> (<delta>)
```

## Final Summary Template

Print when the loop ends (user interrupt, iteration limit, or goal achieved):

```
=== Autoresearch Complete (<iterations_run>/<total_or_∞> iterations) ===
Baseline: <baseline_metric> → Final: <best_metric> (<delta_from_baseline>, <percent_change>%)
Keeps: <count> | Discards: <count> | Crashes: <count>
Best iteration: #<N> — <description>

Top 5 improvements:
  1. iter <N>: <description> (<delta>)
  2. iter <N>: <description> (<delta>)
  3. iter <N>: <description> (<delta>)
  4. iter <N>: <description> (<delta>)
  5. iter <N>: <description> (<delta>)

Run `git log --oneline` on branch autoresearch/<tag> to see the full experiment history.
```

## Notes

- The results file should be added to `.gitignore`. It's a working file for the agent.
- The delta column is always relative to the BASELINE (iteration 0), not the previous iteration.
- For "lower is better" metrics, a negative delta is an improvement.
- For "higher is better" metrics, a positive delta is an improvement.
