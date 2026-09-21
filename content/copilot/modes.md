# Task-local execution modes

Choose from the current invocation only. Without `--mode`, use `balanced`;
never inherit a previous task's mode. Modes change how Hydra spends effort,
not the main model, permissions, acceptance criteria or native session
settings. All modes follow the same [quality floor](hydra-quality.md) and
[context continuity policy](hydra-continuity.md).

| Mode | Priority | Default concurrent workers / total dispatches |
|---|---|---|
| `turbo` | Speed first; costs more | 8 / 32 |
| `balanced` | Quality, speed and cost together | 2 / 6 |
| `economy` | Efficient changes to established designs | 1 / 3 |

Ceilings are instruction-level guidance, not measured host limits: they are
bounded by the Copilot plan's `subagents.maxConcurrency` on usage-based
billing (cap 32) — the lower of the two wins. The 2/6 and 1/3 defaults are
Copilot heuristics, not measured optima.

For multiple substantial, independent subsystems, Balanced can expand to
4 concurrent subagents and 12 total dispatches when parallel progress repays
overhead. Do not expand Economy automatically or silently switch it to
Turbo. All modes allow at most two improvement rounds, inside the same
dispatch budget. Ceilings are not targets; a simple task can use zero
workers in any mode.

## Resolve the policy before dispatch

Use the installed helper without passing task text to a shell:

`node <skill-root>/scripts/hydra-control.js mode <turbo|balanced|economy>`

`mode` alone returns Balanced. For justified expanded Balanced work, append
`--expanded` to the helper call. The helper prints an inspectable policy and
writes no state; it neither starts workers nor changes the active CLI mode.
Its `servicePolicy` applies unchanged across all modes and worker-limit
overrides. Follow [shared-service coordination](hydra-services.md); Turbo
increases useful agent work, not the load on a throttled MCP.

The user may explicitly override ceilings with leading task modifiers
`--max-agents N` and `--max-dispatches N` — requested instruction-level
ceilings, not verified host capacity. Pass only validated numeric values to
the same helper: positive safe integers, with concurrency never exceeding
total dispatches. Invalid or duplicate options are errors, never permission
to guess a budget. Respect smaller host/user limits: the effective ceiling
is no higher than both the resolved policy and applicable host limits.

Record the mode, requested ceilings and main/worker responsibilities in the
task ledger before substantive dispatch. Count every advisor, scan, retry
and escalation; do not reset counters when replanning.

## Task-fit model routing

Choose the worker model for the actual unit of work, not its role name or
the previous dispatch. The catalogue's `modelSelection: task-fit` and tier
hints are instructions, not a model-discovery service or automatic router.

1. Identify the unit's complexity, uncertainty, risk, required context,
   modalities, tools and output constraints before selecting a model.
2. Use the current host's exposed, policy-allowed models and verified
   capability/cost information. Respect user pins, exclusions and budgets.
   Do not invent availability, rankings, price equivalence or reasoning depth
   from names. If no suitable worker can be justified, handle the unit in the
   selected main agent and disclose the limitation.
3. Fit the mode: Turbo favors capable workers and useful parallel speed;
   Balanced trades capability, latency and cost; Economy chooses inexpensive
   workers only when they meet the unit's requirements. High-risk decisions
   and final acceptance stay with the main agent in every mode.
4. Pass the selected model ID explicitly when the native dispatch tool
   supports it; omitting the field can inherit a fixed host default. Record
   the unit, requested model and brief selection reason in the task ledger.
   Report actual models only when exposed; requested is not proof of used.
5. Reassess when scope or evidence changes. If a model is rejected, report
   the substitution and reselect for the same requirements within budget.
   Never silently ignore a user pin or replace a complex unit with an
   incapable cheaper worker; work directly or explain the blocker instead.

The same model may legitimately fit multiple units. Do not rotate models
just for variety, or probe paid models solely to compare names. Reuse valid
host capability information without duplicating discovery for every worker.
This policy never changes `/model`, global `/subagents` settings or workers
that are already running; it does not guarantee a mixed-model roster.

## Turbo

Keep substantive reasoning and final acceptance with the selected main
model. Use strong, current, capable workers for independent implementation
and complex review; smaller workers still handle mechanical operations.
Run implementation, tests and independent reviews in parallel on disjoint
or stable scopes. Cost is secondary, not unlimited: existing user/host
limits and release gates still apply.

## Balanced

Retain Hydra's cost/speed/context tradeoff. Delegate when parallel
progress, a suitable cheaper model or context isolation repays dispatch
overhead. Research only decisions that could materially change the
solution; reuse settled facts. Preserve the selected main agent's difficult
reasoning and acceptance responsibilities.

## Economy

Prefer existing designs, contracts and available project evidence. Minimize
optional research, extra review passes and cosmetic polish. Economy never
waives required checks or accepts known serious defects — security,
authorization, data integrity and deployment gates remain. High-risk logic
stays with the main agent; if a needed review cannot fit the budget, review
directly with a disclosed fallback or report blocked work.

## Large workloads are queues, not unlimited swarms

Schedule only ready independent units with exclusive write ownership, in
bounded waves. Even explicit ceilings such as 100 workers / 1000 dispatches
are requests, not verified host capacity — this integration adds no
massive-swarm scheduler or recursive/factory dispatch.

Report actual mode, models, worker counts, checks and limits; never promise
a speedup solely from a larger team.
