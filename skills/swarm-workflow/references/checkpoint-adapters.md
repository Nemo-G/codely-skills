# Recipe: an outer checkpoint DAG with inner agent teams

Use when the user's task needs multiple issues/checkpoints, shared progress and
verified code integration. This recipe reflects an existing deployment's use of
Codely's public interfaces: an outer command graph, inner role teams, exact-SHA
verification and serialized integration. It does not depend on that deployment's
chat gateway, slash commands, paths, project IDs or models.

## What is built in, and what the task supplies

Swarm supplies graph validation, durable admission, worker/task ownership,
conditions, messages and recovery controls. Control-plane observes those same
facts, persists verifiable incident facts and notifies the coordinator; it does
not choose or run a recovery strategy on its own. Neither component knows the meaning of
an issue, an acceptance test or a feature branch.

The task supplies a checkpoint plan, Git/issue adapter, verifier and acceptance
receipt. A manifest is input to **that adapter**; it is not accepted directly by
`codely swarm workflow run`. Convert it to a supported WorkflowDefinition first.
There is no built-in `codely swarm-feature` command in this contract. Discover an
installed application extension before using any extra command. If no adapter
exists, use explicit stages and authorized Git operations, or implement/test the
adapter within the user's requested scope; do not claim configuration alone
provides automatic feature integration.

```text
coordinating agent / application
  └─ frozen assignment + outer WorkflowDefinition
      └─ core Swarm claims a ready checkpoint command
          └─ task-owned adapter
              ├─ prepare isolated candidate from accepted predecessor SHA
              ├─ admit/observe inner analyze → implement → review team
              ├─ run deterministic candidate checks
              └─ integrate, verify resulting commit, save receipt, exit 0
                  └─ outer Swarm completes node and releases successors

Control-plane observes outer and inner teams in the same Swarm root.
```

**This recipe requires Control-plane**: it coordinates an outer run and inner
teams/runs, and the adapter may outlive its initiating session. This follows
[the explicit checklist](../SKILL.md#decide-whether-control-plane-is-required),
not a subjective judgment that an issue DAG is "complex". Verify the monitor and
outer team before admission, then confirm each admitted outer/inner run in its
snapshot. If required coverage is unavailable, keep new work out of admission.

This is explicit application composition, not a native `subworkflow` chart
field. The outer adapter stays alive until its checkpoint has a terminal result;
returning exit 0 after launching an inner team incorrectly releases successors.
Use ordinary agent nodes instead when this extra business lifecycle is not needed.

## Configure the two layers

1. Establish the persistent Control-plane and shared root, then freeze checkpoint
   IDs, dependencies, acceptance commands, scope, candidate
   base, feature target, final target and authorized remote actions. Do not infer
   edge direction from generic “related issue” links. Exclude reference-only and
   deferred work explicitly.
2. Copy [checkpoint-dag.json](../examples/checkpoint-dag.json). It demonstrates
   `1 → 2 → 3 → {4,5} → 6` with six separate members and concurrency two. The
   outer members use a stable controller cwd; their adapter argv names a frozen
   plan and one checkpoint. They do not share a product-editing branch.
3. Provide `CHECKPOINT_ADAPTER`, `PLAN_FILE`, `CONTROL_WORKSPACE`, `NODE` and
   `CODELY_ENTRY` as described in [complex-workflows.md](complex-workflows.md).
   Its 13-hour command deadline is an example for an adapter with at most
   12 hours of owned work plus cleanup, not a Swarm default. Choose both bounds
   from the task; include repair attempts and cleanup in the total budget.
4. On a ready checkpoint, prepare its unique worktree/branch at the accepted
   base. Only then prepare inner members using that candidate's actual cwd.
   Keep the saved inner chart launches identical to `worker prepare --members`.
   Assign analyst/reviewer read-only roles and an implementer write scope.
5. Inner worker outputs use exact task/run/turn identity and a strict schema.
   Review may produce pass/repair/human; pass alone does not complete the outer
   checkpoint. A bounded repair policy may create a new inner attempt, with
   original authorization and prior findings preserved.
6. The adapter validates candidate identity, runs acceptance, integrates only
   into the designated feature branch, validates the resulting commit and saves
   a receipt before returning exit 0. Failure or unknown state cannot release
   successors. The outer core store remains the only dependency scheduler.

Create and freeze the inner run ID before calling admission. Persist its handle
before observing it. After restart or a lost response, inspect that same run;
do not make a new UUID merely because a CLI call timed out. Same-run recovery is
not the same as a new repair attempt. Stop/cancel must reach all inner work owned
by the adapter; ending the outer command is not proof its children stopped.

## Branches, worktrees and acceptance

For a feature whose user requests an integration branch:

- Start it from the selected target commit. Each checkpoint MR targets the
  feature branch; the final feature-to-main MR is a separate action.
- Use one writer per candidate worktree and a single integrator for the feature
  ref. Do not force-reuse a branch checked out elsewhere or stash/reset an
  unknown/dirty workspace to make it match the plan.
- Parallel checkpoints 4/5 start from the same accepted checkpoint-3 SHA. The
  later integration must incorporate the updated feature head and repeat the
  relevant review/checks for the new candidate. Two green branches do not prove
  their combination is green.
- Verify workspace package resolution in both candidate and post-merge test
  trees. A wholesale `node_modules` link may resolve local workspace packages
  into the main checkout and test old code. Share external caches if appropriate,
  but build local packages against the candidate; initialize only needed submodules.
- Record `checkpoint`, plan revision, attempt/run IDs, candidate/base/reviewed
  SHA, actual integrated SHA, commands/exit codes and evidence paths/digests.
  Test logs and reviewer text are evidence, not permission to widen task scope.
- Recheck remote source/target HEAD before integration and actual ancestry/tree
  afterwards. A lost merge response is reconciled before repeating the action.
  Acceptance is not published while the integrated commit is unverified.
- An issue being closed or an MR being approved is not an acceptance receipt.
  Project-specific closing and final merge policies remain explicit user scope.

## Share progress without a second task store

Give each inner team a read-only snapshot containing the plan revision,
checkpoint DAG, predecessor receipts, accepted base SHA, peer state/MR/evidence
references and blocked reasons. The adapter's snapshot is a projection of core
execution state plus Git/verification facts; it is not permission to edit Swarm
task files or manually set a node completed.

Use `inputsFrom` for committed predecessor results within one chart. For
cross-team business data, use the adapter's explicit artifact reference/query
contract. `swarm send --team <team> --member <member> --kind work --message ...`
can notify a member to refresh a snapshot; verify flags with help. ACK proves
message handling at its documented boundary, not that an issue passed acceptance.
Do not broadcast whole transcripts or use messages to bypass dependencies.

Keep execution and business states distinct: admitted, running, review_passed,
verification_failed, integrating, accepted. A peer's integration may invalidate
a candidate base; notify the peer and require revalidation, not silent file edits
in its worktree. A later revision must explicitly preserve still-running siblings
or stop/drain them before replacement, so repairing checkpoint 4 does not launch
a second copy of live checkpoint 5.

## Check the adapter before unattended use

Exercise the real CLI/command path with local temporary Git repositories and
controlled worker/service fixtures before live issue implementation:

| Scenario                                 | Observable acceptance                                                                      |
| ---------------------------------------- | ------------------------------------------------------------------------------------------ |
| Two independent checkpoints              | Separate worktrees and ready members; same accepted starting commit                        |
| Review pass, test failure                | Nonzero outer command; successor remains blocked                                           |
| Long inner team                          | Explicit outer deadline; no accidental 30-second termination; bounded cleanup              |
| Start or merge response lost             | Original run/MR inspected; no duplicate implementation or merge                            |
| Integration changes candidate SHA        | Old review/check result cannot authorize the new candidate                                 |
| Stop during verify/integrate             | No new admission after revocation; uncertain applied effects reported                      |
| Coordinator/monitor restarts             | Frozen identities recovered; no replay of committed results                                |
| Progress notification lost/duplicated    | Snapshot still reconstructable; dependency state unchanged                                 |
| Parallel sibling continues during repair | Same live attempt preserved, not re-dispatched                                             |
| Final checkpoint succeeds                | Final delivery evidence exists; deferred work and unauthorized main merge remain untouched |

Report which of these were actually exercised. Schema validation alone proves
neither subprocess readiness nor role quality. If a required behavior cannot be
expressed through installed public interfaces, report the exact missing seam;
do not direct a normal agent to inspect internal source or mutate runtime records.
