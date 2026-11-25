# Configure complex work through public contracts

Choose the smallest shape that represents the user's work. Roles, checks and
iteration budgets come from that task, not from a fixed mandatory review process.
For installation/launch setup, use [agent-system-deployment.md](agent-system-deployment.md).
Apply [the Control-plane checklist](../SKILL.md#decide-whether-control-plane-is-required)
before executing a recipe. A bounded single-team graph with continuous controller
attachment can use the CLI alone; outer/inner teams, background handoff,
continued maintenance or requested monitoring require Control-plane.

## Select a recipe

| Need                                       | Shape                                                                    | What to verify                                                                 |
| ------------------------------------------ | ------------------------------------------------------------------------ | ------------------------------------------------------------------------------ |
| Independent collaboration                  | Team messages and shared tasks, no chart required                        | Ownership, delivery ACK and explicit result                                    |
| Ordered stages                             | `dependsOn` plus `inputsFrom`                                            | Successor uses committed predecessor output                                    |
| Parallel analysis or separate write scopes | Different members, independent nodes, bounded `maxConcurrency`, AND join | Each branch is actually ready; no shared file/build races                      |
| Assessment followed by execution           | Structured `resultSchema`, `when`, `pauseOn`                             | Process completion and business pass remain distinct                           |
| User-bounded review/repair                 | Unique nodes for each authorized round                                   | Pass skips repair; exhausted repair/human decision pauses                      |
| Multi-checkpoint code delivery             | Outer command DAG plus task-owned acceptance adapter                     | [Checkpoint recipe](checkpoint-adapters.md), exact base/review/integrated SHAs |

## Materialize the complete examples

Bundled examples are full WorkflowDefinitions, **templates rather than ready
executions**:

- [parallel-verification.json](../examples/parallel-verification.json): two
  read-only analyses, review, deterministic command gate, final handoff.
- [bounded-repair.json](../examples/bounded-repair.json): initial review and one
  repair/review attempt. Use only when that bound matches the task.
- [checkpoint-dag.json](../examples/checkpoint-dag.json): the outer graph for
  an application adapter owning each checkpoint's internal work.

Copy the chosen file into the task's run directory. Replace each `{{...}}`
token as a JSON string value, not by shell interpolation:

| Token                              | Required value                                                                                       |
| ---------------------------------- | ---------------------------------------------------------------------------------------------------- |
| `NODE` / `CODELY_ENTRY`            | Verified absolute Node executable / installed CLI bundle; adjust launch for standalone distributions |
| `WORKSPACE`                        | Existing workspace containing the saved user assignment; separate worktrees for concurrent writers   |
| `VERIFY_SCRIPT`                    | Absolute task-specific verifier, prepared and tested before admission; not supplied by Codely        |
| `BASE_SHA`                         | Exact candidate SHA the verifier must check; regenerate the unadmitted input when it changes         |
| `CONTROL_WORKSPACE`                | Stable directory for the outer command carriers                                                      |
| `CHECKPOINT_ADAPTER` / `PLAN_FILE` | Tested task-owned adapter and its frozen plan; see checkpoint reference                              |

Edit the example's prompts to name the actual assignment, scopes and evidence
paths. Keep the exact task/run/turn identity when submitting with `swarm_task`.
`skillRef` is a label; it does not load a role file. Examples omit model selection
to use the configured default; requested models belong in verified launch args.
Member IDs use hyphens so they also work as CLI member names.

The parallel example is read-only until its task-provided verifier runs. The
bounded-repair example contains an editing assignment: create the authorized
candidate worktree first, serialize edits, and ensure the reviewer inspects that
same candidate. Do not apply a read-only shared-directory configuration to
parallel implementers.

Run the public sequence after preparing members from the final launches. Where
the checklist requires Control-plane, verify health and the prepared team before
`run`, then verify the admitted run snapshot using the deployment reference:

```text
codely swarm workflow validate --file <materialized.json> --json
codely swarm worker prepare --team <team> --members <JSON-array-derived-from-chart>
codely swarm worker status --team <team>
codely swarm workflow plan --team <team> --file <materialized.json> --json
codely swarm workflow run --team <team> --file <materialized.json> --run-id <saved-uuid-v4>
codely swarm workflow status --team <team> --run-id <saved-uuid-v4> --limit 100
```

The expected initial frontier of the parallel example is `inspect-api` and
`inspect-storage`; review cannot start until both complete. A `pass` permits
`verify`. `repair` or `human` pauses before verification. Handoff requires the
verification node to complete successfully. The example proves scheduling;
business acceptance depends on a real verifier for the task.

## Commands and acceptance are not model claims

`command` runs executable + args with no shell in the run's recorded workspace
directory: the durable member's workspace dir, normally the `cwd` its worker was
launched with, not the chart's `workspaceRef` label. Use absolute paths or
confirm that directory first. Use an explicit interpreter for scripts. Set
`timeoutMs`: the current
process default for an omitted value is 30 seconds. Choose a deadline covering
the work and its cleanup; a long-lived adapter needs a larger outer deadline
than its own bounded internal work.

The actual command result schema is `{exitCode, stdout, stderr}`. A nonzero exit,
signal, timeout or cancellation is not a successful gate. JSON emitted to stdout
stays a string; declaring `{verdict: ...}` as the top-level resultSchema does not
parse it. `inputsFrom` does not substitute command args/env. Pass a frozen input
file explicitly, or use a task-owned adapter that reads and validates it.

The verifier in the parallel template must at least enforce the admitted
candidate identity and the user-selected checks, retain bounded logs/artifacts,
and exit nonzero when acceptance is missing. Keep the verifier definition outside
the implementer's writable scope; checking candidate code is not permission for
candidate code to replace its gate. A command that merely prints `pass` is not
verification. Do not claim a shipped Codely verifier exists at the template path.

## Bounded repair and joins

The repair template has `review-first → repair → review-final` with repair
conditional on the first verdict. Initial pass skips the later nodes; inspect
that committed verdict when reporting success. Final repair/human pauses. There
is deliberately no common final node that depends on both an early-pass branch
and a skipped repair branch: `dependsOn` is AND, not OR, and skipped dependencies
do not satisfy an ordinary join.

A chart cannot contain cycles. Expand a user-specified bounded number of rounds
with unique IDs; do not invent an endless retry loop. Do not replay a committed
`pauseOn` business decision with `resume`. A changed candidate/decision needs a
successor definition/run or an explicit task-owned repair adapter. If the user
requires a single chart and the requested join/loop cannot be represented, report
that limitation instead of silently splitting the work.

## Recovery by observable condition

| Observation                                  | Next public action                                                                                                            |
| -------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------- |
| Missing skill                                | Verify Swarm and skills are enabled; inspect `/skills` catalog; do not invent a `swarm-workflow` CLI command                  |
| Unknown chart key                            | Re-read `workflow schema --json`, repair the source definition, validate again                                                |
| `resultSchema` cannot be executed            | `workflow validate` reports the node, field path and failing keyword. Fix the schema, then validate again.                     |
| `plan` not runnable                          | Inspect missing members and prepared launches; do not fabricate roster rows                                                   |
| Member attached but work remains pending     | Check worker readiness, ownership and dependencies; attached alone is insufficient                                            |
| Launch conflict                              | Inspect open work, stop/release only the old owned tenure, then prepare; never overwrite its frozen launch                    |
| Command exits around 30 seconds              | Check its declared timeout; fix a new definition/attempt rather than treating omitted timeout as unlimited                    |
| Command result schema rejects stdout JSON    | Use the command envelope; validate an explicit business artifact in the adapter                                               |
| Completed execution but repair/human verdict | Report the business state and respect its gate; do not mark delivery accepted                                                 |
| Response lost during admission               | Query the original team/run and compare frozen definition before retrying with the same identity                              |
| Paused/failed node                           | Inspect committed result and effect state before choosing resume/retry; the controls act on a run, not an assumed single node |
| Monitor cannot see known team                | Check shared root/home and team ID before restarting workers                                                                  |

If required Control-plane coverage is unavailable, use CLI status for diagnosis
and reconciliation, not to start new unmonitored work. Already admitted workers
may continue; an outage does not itself cancel the graph. Restore observation
before new admissions/resumption, and follow the task's explicit outage policy
for any pause/cancel of existing execution.

Control commands, after checking installed help:

```text
codely swarm workflow pause --team <team> --run-id <run> --operation-id <uuid>
codely swarm workflow resume --team <team> --run-id <run>
codely swarm workflow cancel --team <team> --run-id <run> --operation-id <uuid>
codely swarm workflow retry --team <team> --run-id <run>
```

Pause/cancel can request control of active turns. When the leader is stopped,
`workflow cancel` cancels pending work and returns `activeRequested: 0` while
active peers keep running. Inspect returned counts, conflicts and subsequent
settlement. Retry/resume are run-wide: do not assume an unsupported `--issue` or
`--node` selector. Application-level “pause between phases” is a separate adapter
behavior and must be named as such. Uncertain external effects require
reconciliation, not a fresh run that repeats them.
