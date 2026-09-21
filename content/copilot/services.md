# Shared MCP services and slow dependencies

Apply automatically when a Hydra task depends on MCPs or other remote tools.
Optimize completed useful work, not agent count. Worker concurrency and service
request concurrency are separate budgets in every mode, including Turbo.

## Split work around the bottleneck

Decompose substantial goals into ready units with dependencies and acceptance
criteria without another user prompt. Group tiny steps; prioritize work that
unblocks other units. Keep remote retrieval separate from analysis and edits.
Independent code, research or review can continue while a data branch waits;
work needing that data must not invent it or ship without required evidence.

Assign one fetch owner per shared provider/account/tenant quota scope. Other
workers request data from that owner, not directly from the same service.
Pass the service restrictions to every worker. A new agent, endpoint alias or
rephrased query does not create a new quota. Cross-session quotas may also be
shared: this task-local policy cannot coordinate other sessions automatically.

Deduplicate identical in-flight reads by tool + canonical arguments + authorized
scope; share results only within that scope. Reuse completed results when their
timestamp and freshness meet the task; never reuse a failure as an empty success.
Invalidate affected data after mutations. Keep only necessary task data in the
session, not a global incident cache, full response dump or public artifact.

Inspect the current tool schema first. Prefer supported bulk queries, narrow
filters/time windows and bounded pages; do not fan out a single call per record
when the tool supports a batch. Fetch expensive full context/discussion only for
needed records. Follow continuation tokens, count pages against the shared
budget, and report incomplete coverage. Never invent batching or timeout fields.
This applies to incident systems such as IcM as well as other rate-limited MCPs.

## Independent service budget

`hydra-control.js mode` includes `servicePolicy`; `service-policy` inspects it.
Defaults are conservative Hydra heuristics, not service SLAs or enforced limits:

- One in-flight request per shared quota scope, irrespective of worker count.
- At most three attempts per read (initial plus two retries).
- At most twelve requests across a retrieval branch: pages, details and retries.
- A 300-second elapsed branch budget; report slow retrieval after 60 seconds.

Use stricter user/host/service limits. A request for more workers does not extend
these limits. For a legitimate larger collection, agree an explicit bounded
continuation rather than silently resetting counts or running a retry loop.
The helper exposes these defaults; a different authorized service budget needs
explicit ledger tracking, not fabricated helper options.

## Throttling, retries and unresolved calls

On observed throttling (for example 429 or an explicit rate-limit result), pause
the whole quota group, not just one worker. Honor server `Retry-After` as a minimum;
normalize an exposed seconds/date value using its receipt time, never shorten it.
For completed failed reads, use bounded exponential backoff plus small jitter.
If the next wait does not fit the remaining budget, stop that branch and report
what is missing. Do not probe alternate endpoints/accounts to evade throttling.

The local adviser accepts metadata only:

`node <skill-root>/scripts/hydra-control.js service-plan <pending|throttled|transient|permanent> <read|write> <attempts> <requests> <elapsed-seconds> [retry-after-seconds]`

`attempts` includes the initial call; `requests` counts the whole branch.
`elapsed-seconds` measures the whole branch, not the latest call's duration.
Do not label an individual call slow based only on earlier requests' time. Classify
from observed tool status, not guesses. Transient means a completed failed read;
unknown completion stays `pending`. Omit retry-after only when absent; malformed
or ambiguous server advice is not permission to retry early. The helper proposes
a wait range; choose jitter within it and recheck elapsed time and quota before
retrying. It does not sleep, send a request, hold a lock or intercept MCP traffic.

Never duplicate a pending call, even when the budget expires. An unresolved
mutation must be reconciled, not replayed automatically; permission/auth/validation
failures need action, not more retries. Check whether the host/service already
retries internally before adding another layer. Include that time in the budget.

If native background retrieval/status/cancellation is supported, use it only
with real independent work. When the host returns control, report slow or
throttled state and any useful partial results instead of hiding the entire task.
After the deadline, stop new requests and mark the data branch blocked; an
outstanding call may still be running. Cancel only through a supported control
and confirm termination before replacing its owner. Do not kill shared MCP
processes, restart another session, or claim a timeout you did not enforce.

A blocking synchronous MCP can prevent the agent from issuing progress messages
or cancellation until the host returns control. This skill cannot fix that
transport limitation or promise a five-minute return. Report the limitation
honestly; do not mistake "still running" for useful progress or confirmed throttling.

## Preserve progress and report coverage

Checkpoint request keys, owner IDs, timestamps, completed/pending items, attempts,
branch request count, cooldown and remaining work. Resume from that state, not
from a fresh budget. Treat unknown elapsed time or in-flight status as unresolved.
Keep sensitive incident content and credentials out of the control helper.

Return available results with their source/time, missing coverage, observed
failure or slow-call state, and the next safe action. Required missing data
blocks completion/deployment; partial results are never described as complete.
