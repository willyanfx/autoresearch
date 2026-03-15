# Core Principles of Autoresearch

These 7 principles are extracted from Karpathy's autoresearch and generalized to any domain.

## 1. Constraint = Enabler

Autonomy succeeds through intentional constraint, not despite it. A bounded scope that fits in
the agent's context, a fixed iteration cost, and a single mechanical success criterion are what
make autonomous operation possible. Without constraints, the agent wanders.

## 2. Separate Strategy from Tactics

Humans set direction (WHAT to improve). Agents execute iterations (HOW to improve it). The user
chooses the goal, scope, and metric. The agent chooses which specific changes to try and in what
order. This division of labor is what makes the pattern scale.

## 3. Metrics Must Be Mechanical

If you can't verify with a command, you can't iterate autonomously. Good metrics are numbers
produced by running a command. Bad metrics are subjective assessments like "looks better" or
"seems cleaner." The entire feedback loop depends on mechanical, repeatable measurement.

## 4. Verification Must Be Fast

Cheap iteration enables bold exploration and many experiments. Expensive iteration forces
conservative choices and few experiments. Aim for verification under 10 seconds. At 10s per
iteration, you get 360 experiments per hour. At 5 minutes per iteration, you get 12.

## 5. Iteration Cost Shapes Behavior

The cost of each experiment determines the agent's strategy. When iterations are cheap, the agent
can afford to try radical ideas — most will fail, but the occasional success compounds. When
iterations are expensive, the agent should be more conservative and targeted.

## 6. Git as Memory

Every successful change is committed. Failures are reverted. This creates a permanent record of
what worked and what didn't, enables stacking wins on top of each other, allows pattern learning
from git history, and makes it easy for humans to review what happened.

## 7. Honest Limitations

If the agent hits a wall — missing permissions, external dependency, needs human judgment, or
diminishing returns — it says so clearly instead of guessing or spinning its wheels. Transparency
about limitations is more valuable than pretending progress.
