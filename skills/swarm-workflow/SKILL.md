---
name: swarm-workflow
description: >-
  Organize Codely agent teams from natural-language goals, roles and constraints.
  Covers flexible team collaboration through messages and shared tasks, and
  Chart-based WorkflowDefinition runs for declared dependencies, structured
  result handoff, editable DAGs and multi-checkpoint delivery.
---

# Swarm Workflow (N1)

`swarm-workflow` is the **skill name**, not a CLI subcommand. In a Codely
session with Swarm enabled, discover it in the skill catalog and load it with
`activate_skill({"name":"swarm-workflow"})` using the available tool schema.
The actual management commands are `codely swarm ...` and
`codely serve control-plane ...`. There is no `codely swarm-workflow` command.
An agent outside Codely can read this skill and its relative resources, then
operate those same public commands. It does not need the Codely source tree or
an application-specific slash command.

Accept a short request naming the goal, roles, models and constraints. Own the
discovery and execution procedure; do not ask the user to supply a tool manual.
**Never guess** model ids, auth, tools or chart fields.

## Return results to the requesting session

In Codely, a leader chat can create the team in its own session. From Codex
or Cursor, delegate to a native Codely leader through an ACP client that
keeps its stdio connection open. For unattended work, use the
[supervised leader host](references/agent-system-deployment.md#unattended-leader-and-member-recovery).
Record the requesting session, its return channel, the ACP `sessionId` of
the bound leader session, `teamId` and the managing leader member ID.
Keep the connection open while members work and verify that a later prompt in
the same leader session can read the team. If startup stops, record the last
successful ACP request and the next request's actual error. For example, if
`initialize` succeeds and `session/new` cannot write its project state, the
outer client has no leader session to dispatch from.

For unattended work, verify the host's isolation and use the same account,
workspace, state directory and tool policy for preflight and execution.
Start the intended client, retain its successful `session/new` receipt, and
read the team back in that session. Verify the first required operation
through the role that will perform it. Check the carrier and the external
member separately, including the member's login, permitted workspace, tools
and MCP access. After a restart, verify the same identity and permitted
scope from the running processes. Keep credentials out of the receipts.
If access is denied, record the failing operation, path or service, and
error; leave that action pending and continue independent tasks. Publishing
guidance and changing user availability still require the user's confirmation
of the specific change. An offline decision uses the current recorded grant.

On return, use the original leader session to read the durable task or run
result and send those references back to the requesting session. Give the
user `codely swarm attach --team <team-name>` for a terminal leader, or the
ACP `session/load` identity and working directory, to reopen the same conversation.

For “Bob must review every release”, declare the delivery, Bob's review,
rework and publication dependencies in a Chart. Record the same `runId`
for Bob returning v1, Alice delivering v2, Bob approving v2 and the
publication that consumes that approval. Check that the v1 return left
publication unexecuted. On re-entry, query the original `runId` with
`swarm workflow status`. Follow a running Chart in place. If it is paused
and continuation is authorized, resume the same `runId`. If admission has
not happened yet, continue validating the saved Chart before dispatch.
Report the admission error if it persists.

For “Alice may ask Bob”, follow [the ordinary review handoff](#review-a-versioned-ordinary-delivery).
Alice decides whether to open a review. Give Alice and Bob the task ID and
artifact revision to cite, and agree who will open any rework before dispatch.
The leader reads the receipts when the requester returns.

Capture the path and digest of the skill the initiating client actually
loaded. Retain Codely's `activate_skill` result or the Codex/Cursor file
read of the deployed `swarm-workflow/SKILL.md`. For unattended recovery
retain the linked recovery recipe's digest from the same installation.

## Choose how to dispatch work

Flexible collaboration uses team messages and shared tasks. A Workflow declares
steps in a saved Chart (`WorkflowDefinition`). Each admitted run has a durable
`runId`. Preserve the user's choice. When it is open, suggest an option based
on the handoff, tracking and execution behavior the task calls for.

For example, `reader` reads README and `reviewer` verifies the findings.

| Mode | Scenario | Dispatch and evidence |
| --- | --- | --- |
| Flexible | Named recipients coordinate the handoff directly. | Send `work` with `swarm_send` to a persistent member or explicitly activate a local headless recipient after sending. The receipt confirms mailbox acceptance. Check consumption through `swarm events` and completion through the reviewer's `result` reply. |
| Flexible | The team needs a shared backlog with owners and claim ordering. | Create shared tasks with `swarm_task` for persistent workers and use `blocked_by` to gate claiming. Agree where to read the summary. Query ownership, blockers and completion by `task_id` with `swarm_task status` or `codely swarm task status`. |
| Workflow | The team needs a repeatable run with declared ordering and same-run structured output handoff. | Define `dependsOn` and `inputsFrom` in the Chart. Query node settlement, committed results and input lineage by `runId` with `codely swarm workflow status`. |

## Query team state from an external agent

An agent already joined to the team can query the same durable facts through the
existing Gateway MCP endpoint (`/v1/swarm/mcp`) with its own member bearer token.
No team id or member id is supplied in the arguments: the token fixes the team
scope and is checked against current membership on every call.

| Tool | What it reads |
| --- | --- |
| `team_status` | Token team identity, caller role/state and the names of the available queries. |
| `members_list` | Bounded member page: id, name, optional duty description, state and last heartbeat. |
| `tasks_list` | Bounded task page: durable `status`, recorded `workflowStatus` when that field exists, owner, declared `blockedBy` dependencies, ordinary/Chart kind and whether result evidence exists. |
| `task_get` | One task by id: the same status fields, plus a committed result and artifact references when present. Missing evidence is explicit. |

Follow `nextCursor` until it is null to cover the whole list. A page states its
own limit, count and `partial` flag. `page.partial` is true only when this
read's store diagnostics skipped an unreadable record. The page still returns
the readable entries and the cursor. Diagnostics from an earlier request or
another team do not make this page partial, and a partial empty page is not a
healthy empty team. Heartbeat time is not task progress, and a committed result
is not review approval. `blockedBy` is the declared dependency list; judge
whether the task is currently blocked by checking those tasks' status. `status`
stays the durable task status. `workflowStatus` is returned only when the
record has that field, so a failed ordinary task can stay `status: pending`
with `workflowStatus: failed`. A missing `workflowStatus` is reported as
missing and is not inferred to be a legacy Chart. Chart association stays on
the Chart record. Artifact evidence is `present` only when the committed
result contains at least one artifact reference. An empty artifact list still
keeps `committedRecordId` and does not report artifacts present. Missing
results, undeclared artifacts and legacy Chart records are reported as facts
rather than inferred. Each response declares a bounded consistency scope:
fields come from the records read for that answer,
and cross-record summaries or pages may span different write moments. Re-read
before making a decision. These queries never claim, settle, reconcile or
acknowledge anything; task writes and agreements are separate capabilities.

## Deliver ordinary work through MCP

Use the joined member's Gateway connection and check the advertised tools.
Both native `swarm_task` and external members work on the same ordinary task.

1. Create it with `task_create`, supplying a subject, a description and a
   stable `requestId`. Keep the returned `taskId` for the other member.
2. The worker calls `task_claim` with that ID. Save the returned `taskId`,
   `taskSequence`, `workflowRunId`, `workflowTurnId` and
   `workflowTurnSequence`. These five fields identify this specific claim.
3. Call `task_complete` with those fields, a stable delivery `requestId`
   and the result. If a response is lost, retry with the same request ID and
   payload. With `task_get`, read the ordinary delivery's `committedRecordId`
   at `ordinary.committedRecordId` and its declared artifact at `ordinary.artifact`.
4. After processing an item from `messages_read`, call
   `messages_ack({messageId: item.id, sequence: item.sequence})`. The
   `messages_ack` receipt confirms
   consumption of that message. A Chart assignee settles its node separately
   with `result_submit` and the assignment's own proof.

## Read and publish team guidance

On joining the team, resuming a session or returning from context compaction,
read the team's `guidanceVersion` from status. Version zero means the team
has no published guidance. For a positive version, fetch the full text with
the native `swarm_manage` action `get_guidance` or Gateway MCP
`team_guidance_get`. Keep the returned version with the text. On later status
checks, refresh the text when the version changes.

Members may propose guidance. The native leader publishes only after the
user explicitly confirms the exact text to save or update in the leader chat.
Call `publish_guidance` with that `text` and the `expected_version` read
from the team. If another publication wins first, read the new version and
reconfirm the proposed update. Report the version actually saved.
If the leader cannot obtain the user's one-time confirmation for a proposed
version, keep that version pending. Continue independent work under the
last genuinely confirmed guidance and rules. An unattended agent's general
tool approval cannot stand in for the user's publication decision.
The guidance is advisory. A change in review
instructions applies to subsequent work; record any change to an in-flight
task explicitly. Retain the delivery and review references already committed.

## Apply ordinary-task rules

Read the current `ordinaryPolicy` through `get_guidance` or
`team_guidance_get`. Guidance text stays advisory. To enforce a change, the
user confirms the complete text and structured rules in the leader chat.
Use stable member IDs in `memberDuties`, task kinds in `taskEligibility`,
and the dispatcher's member ID plus any conditions in `reassignGrants`.
An update containing only text retains the last confirmed rules. Submit
empty rule lists to revoke them.

On creation, use `taskKind` in MCP or `task_kind` in native `swarm_task`.
Check the live task and applicable policy before handoff. MCP members use
`task_reassign` with a stable request ID, the target member and a reason;
for an owned turn, read `claimIdentity` from the dispatcher's own `task_get`
response and pass its four fields with the task ID. Re-read the task if
the turn has changed. A repeated request must describe the same handoff.

For unassigned, claimable ordinary work, the supervisor uses the team's
existing wake policy and remaining action budget. A pending wake reservation
identifies a candidate and expires if the task remains unclaimed. Read the
actual holder from the task: an online eligible member can still
`task_claim` it before the parked member returns. Use the Chart's assignment
and capabilities for work admitted through a Chart.

## Handle a decision while the user is away

Use `swarm_manage status` to read `userAvailability`. With no declaration
the user is present at version 0. The leader records a user's departure or
return with `set_user_availability`, the requested `availability` and the
observed availability `expected_version`. This takes the user's explicit
one-time confirmation. A disconnected client does not declare an absence.

For a delegated ordinary handoff, use `publish_guidance` to present the
full guidance text and replacement rules for confirmation. An
`offlineReassign` entry specifies `trigger: user_declared_away`,
`taskKind`, `toMemberId` and an optional `fromMemberId`. Keep the
corresponding ordinary reassignment grant when the leader is acting for
another holder. To revoke only offline permission, publish the complete
ordinary rules you intend to retain with an empty `offlineReassign` list.
Guidance versions and availability versions are independent.

1. A member uses `swarm_task propose_decision` with a stable UUID in
   `decision_id`, the ordinary `task_id`, `to_member_id`, reason and the
   observed claim's four identity fields. Preserve this ID across restarts.
2. The leader uses `process_decision` with the same task and decision IDs.
   With no applicable authorization, leave it pending and continue independent
   tasks. Processing rechecks the current confirmed rules and the declared
   absence window.
3. On return, page through `list_decisions` and read details with
   `get_decision`. Report `pending` as waiting, `awaiting_confirmation` as
   an attempted operation whose effect still needs checking, and
   `executed` with its actual handoff receipt and authorization basis. An
   executed decision can still lack `acknowledgedAt` after an interruption.
   The leader reuses `process_decision` to reconcile that receipt. Use the
   ordinary task path for new instructions from the returning user.

## Review a versioned ordinary delivery

Read the current team guidance and agree whether this task calls for direct
delivery or independent review. For **direct delivery**, the holder completes
the task and the leader reports the committed delivery. For a review, use
separate ordinary tasks so each decision has its own holder and receipt.

1. **Deliver v1.** The implementer completes the work task with the artifact's
   `repository`, `path`, immutable `revision` and SHA-256 of its actual bytes.
   Native `swarm_task complete` can return `settlementPending`. Wait for
   settlement, then call native `status` with `task_id` and read
   `delivery.committedRecordId`. With MCP read `ordinary.committedRecordId`
   from `task_get`. Keep the task ID and the committed record ID together.
   Use this readback for each delivery and review before reporting its outcome.
2. **Review v1.** When independent review is needed, the implementer opens the review task
   for the reviewer and passes the delivered task ID and artifact revision. The reviewer reads
   the delivered task and verifies that version of the artifact. With MCP,
   `task_get` supplies `taskId`, `sequence` and `ordinary.artifact`.
   In the reviewer's own `task_complete`, set `referencedTask.taskId` to that
   work task and `referencedTask.taskSequence` to its `sequence`. Use the
   reviewer's *own* claim fields at the top level. In native `swarm_task`,
   the corresponding delivery field is `delivery.referenced_task` with
   `task_id` and `task_sequence`. Report the verdict, reviewed revision and
   digest in the review's summary.
3. **Rework v2.** On a return, the reviewer sends the committed verdict directly to the implementer.
   The member responsible for follow-up opens the rework task with the review
   task ID and `committedRecordId`. The implementer cites that review in the
   v2 delivery with the revised artifact and its digest. On settlement,
   read back the structured reference from native `delivery.referencedTask` or
   MCP `ordinary.referencedTask` and verify the cited task and sequence.
   Open a new review task and cite the v2 **work task** in its receipt.
   Preserve the earlier v1 delivery and decision. Report approval only after
   the review receipt identifies the same task and artifact version.
4. **Recover the leader's view.** Use `codely swarm attach --team <team>` for
   a terminal leader or ACP `session/load` for the existing bound session.
   Read the current tasks by ID. Use native `status.owner` or MCP
   `ownerMemberId` to identify the holder. Report the artifact revision,
   referenced work task and committed review decision. A separate direct
   delivery has only its delivery receipt.

A Chart reviewer may cite an ordinary investigation's task and receipt in
its result. The Chart assignee still completes the admitted node through its
original assignment and attempt. A Gateway assignee uses `result_submit`.

## Choose the member execution mode

Member execution mode describes how work gets a process. Match it to the
dispatch path.

- Persistent `swarm worker` members keep a running worker between assignments.
- Local `swarm headless` registers a member without a process. A caller
  explicitly activates one ready unit at a time. The member and canonical
  session carry over to the next activation.
- Remote `serve web-ui --swarm-on-demand` starts one ACP agent per accepted
  assignment for Gateway-ready Chart nodes bound to the attached member.
  Mailbox work does not start the agent. Keep ordinary shared tasks out of a
  team with this host because unbound tasks can be offered to it.

See [headless-members.md](references/headless-members.md) for discovery,
verification and recovery for both on-demand paths.

When the user asks for a clickable team DAG, live HTML, Raw Chart editing or
interactive node configuration, read [live-chart-editing.md](references/live-chart-editing.md).
The default HTML/CSS/JS chart template ships once with the installed CLI;
`workflow dashboard` mechanically copies that published template to `--output`
and writes `<output>.binding.json`, and the skill generates only binding
configuration. Never generate per-team business UI, copy implementation code,
or keep a second Chart format or scheduler. The browser composes the existing
Control-plane API in one snapshot adapter: it does not introduce a second
execution source, scan logs, or auto-recover. The global Control-plane page is
an example/overview, not the team's deliverable.

## Decide whether Control-plane is required

For unattended work that must survive a leader process crash, follow the
[leader and member recovery recipe](references/agent-system-deployment.md#unattended-leader-and-member-recovery)
before admission. Identify the independent host that can restore the original
leader session and the execution host that can bring each required member back.
Report automatic recovery as unverified until the deployed skill and those
hosts have completed the crash scenario together.

Do not classify a task by the word "complex", a node count or a duration guess.
**Use a persistent Control-plane for the same Swarm root if any condition holds:**

- The user asks for monitoring, an operational dashboard or continued maintenance.
- Work must continue after the initiating controller process/session detaches or
  exits, rather than remaining attended through terminal settlement.
- The assignment coordinates multiple teams/runs, such as an outer checkpoint
  DAG that creates inner implementation/review teams or successor repair runs.
- The deployment requires unattended incident notification and
  coordinator-driven recovery, unattended scheduled/repeated operation, or
  restart/reconnect recovery as part of its delivery contract.

CLI-only execution is allowed only when **all** apply: one team and one bounded
assignment/run, a live coordinator remains attached until settlement, no continued
monitoring/maintenance or unattended recovery is required, and the coordinator
can observe the terminal outcome directly. A small DAG can satisfy this exception;
a single long-running background node may not. Ending a model turn while its
controller process remains attached is not the same as losing that controller.
If these conditions cannot be established, use Control-plane. Reuse an existing
healthy instance for the root even when starting a separate one is unnecessary.

An attended caller can run local `swarm headless activate` without
Control-plane, wait for the outcome and explicitly start the next unit.
Control-plane does not activate dormant local members. A running remote
on-demand host independently polls the Gateway for Chart assignments bound
to its attached member.

When the decision above requires coverage, reuse or start the monitor before
admitting work and verify three things: its JSON health response, that
`swarm.supervisor.ticks` strictly increases between two reads a few seconds
apart (an HTTP 200/`ok`, a reachable page or a visible snapshot prove
reachability and data visibility only — never that the tick loop is still
running), and the intended team ID. In the bounded CLI-only exception, skip
that step and keep the coordinator attached instead. After admission, verify
that the same run ID is visible in its snapshot. Record the monitor URL, root,
team ID and run ID in the handoff (the CLI-only exception has no monitor URL to
record).
Follow the concrete setup and checks in
[agent-system-deployment.md](references/agent-system-deployment.md#required-control-plane-preflight).
Do not silently fall back to unmonitored execution for tasks that require
coverage. This is the skill's operating requirement; `workflow run` does not
enforce it automatically.

## Natural-language request to a durable Chart

The main session is the natural-language coordinator: it interprets the request
with this skill, writes one authoritative `WorkflowDefinition` JSON file, and records
the original request beside the generated definition. The deterministic boundary
is the existing CLI sequence `validate` → `plan` → `run`; admission computes and
persists the `definitionHash` and `runId`, which are the durable identities for
later status and recovery. A generated JSON file alone is not execution evidence.
After admission, edit the same source through the public Apply operation described
in the live-editing reference. It validates and writes the actual file, updates
unclaimed work, and retains the original snapshot of already-owned attempts.
Saving a JSON file directly is not proof that admitted tasks adopted its changes.

A claimed workflow task stays with its exact owner when the model ends the turn while task-owned background work is still active. `waiting-background` is derived from that claim plus the worker's canonical background activity; it is not a second task status or scheduler. The same member session continues when the existing background completion notification arrives, and exactly one fenced `swarm_task complete` settles the task. Do not rerun the child or replay a committed result. Ending the turn with no task-owned background work and no completion proof still fails closed (`SWARM_TASK_COMPLETION_PROOF_MISSING`). Explicit cancel, stop, a stale fence, background failure, and process crash also stay fail-closed and must not become a later success.

An unattended leader discloses the carrier that is actually attached (for example ACP) and states that this carrier is not a member executor. It does not call `ask_user`. Parent decisions arrive only through the verified task-owned revisioned command channel. When that channel is unavailable, record a durable `needs_human` result instead of a private prompt.

For persistent execution, after confirming workers are ready, the run is admitted, and required
Control-plane visibility is verified, report its identity and monitor URL
briefly and **end the current model turn**. Do not use sleep/status shell loops,
repeated session-log reads, or a goal continuation to poll for progress. The
attached runtime observes durable completion and decision events without model
calls; respond when a meaningful notification or a new user request arrives.
Intermediate Chart nodes advance through the existing task dependencies.

For explicitly activated headless members, follow the activation procedure in
the headless reference instead. Registration establishes configuration, not
process readiness; ready/claim happens inside activation. An admitted Chart
with dormant headless members still needs explicit activations in this release.

Use the existing Control-plane monitor API or `codely swarm workflow status` for
an explicit status request or diagnosis, and send durable member messages when
a decision or handoff is needed. Control-plane does not parse
natural language, regenerate a chart, call a model, or create a second scheduler;
it only observes the durable team/run facts already admitted by the main
session, persists verifiable incident facts, and notifies the leader/coordinator.
Its periodic inspection tick is not a workflow tick: it never starts or resumes
a worker, advances a generation, or re-runs a task on its own. Decide whether,
when and how to recover through the explicit worker/workflow management controls.
A fixed `resume` + `retry` sequence is one sample, not the only universal
strategy or order; choose the action from the durable facts the inspection
preserved and reported.

For stuck diagnosis, follow one evidence chain: current `status` first, then a
bounded `codely swarm events --team <team>` page (or `swarm_task` action
`events` with `task_id` omitted), then `--expand` only when the fact's canonical
task/result/message evidence is needed. CLI, tool and Control-plane read the
same per-team runtime journal; the journal is an action ledger, not a second
task or mailbox store. No new journal event does not mean a member is stuck: it
may be working between observable transitions, so combine the ledger with the
current member/process/task facts. External executors have partial visibility;
report only dispatch/session/exit/result evidence actually observed by the
authority and never infer hidden progress.

### Send a steer and verify its receipt

For a steer sent with `swarm_send`, retain the exact team ID, target member ID,
current tenure/generation, returned `messageId` and mailbox sequence. The send
response is the sender-side acceptance receipt: the durable mailbox write was
accepted. It does not prove that the recipient session has consumed the message
or started the requested work.

Verify the exact message through the per-team journal, paging from the returned
`nextCursor` and expanding canonical references when needed:

    codely swarm events --team <team> --member-id <targetMemberId> --limit 50 --json
    codely swarm events --team <team> --member-id <targetMemberId> --after <journalId:cursorSequence> --limit 50 --expand --json

The mailbox sequence printed by `swarm_send` is not the journal cursor sequence.
Match evidence references to both the mailbox message ID and its sequence; check
the receiver member/tenure/generation on the committed and ACK facts. Read the
canonical recipient conversation record when the ACK reference needs
verification. An ACK proves the message was committed to the recipient's
canonical session. If that same ACK carries `turnEndedAt`, the model turn carrying
the steer also ended. Neither fact proves the requested task completed.

For actionable steers, ask the leader to report against the original `messageId`
and mailbox sequence. If the sender is a team member, the leader can return a
`kind=result` message to that member. For an external sender, use its registered
return channel; do not assume a generic Codely mailbox can reach an outer chat.
The labels below are a requested reply convention, not automatic Swarm states:

- received: identify the matching committed/ACK evidence;
- in_progress: give the claimed task/run ID and owner;
- blocked: name the actual prerequisite or error and supporting evidence;
- completed: provide the canonical task result or artifact reference.

Verify progress and completion against `swarm_task` or `swarm workflow status` and
the durable result. Use the sender's supported return channel for that report;
a detached external CLI or outer chat is not implicitly woken by a mailbox ACK.
Treat a matching journal fact as positive evidence. A missing fact is
inconclusive: the canonical conversation or ACK write can succeed while its
subsequent journal fact is absent. In that case, use the installed version's
read-only exact transcript/ACK lookup. If no such lookup is exposed, report
consumption as unknown and request a correlated task or result receipt; do not
infer failure from a missing journal row or an empty mailbox.
If only accepted or committed is proven, report that phase and leave consumption
pending or unknown until exact recipient evidence is available. An empty inbox,
an idle member or a local coordinator log does not prove consumption. Do not
resend the steer or create a duplicate task until its possible existing side
effects have been checked.

When a run is not terminal, still has pending tasks and no task of it is
running, the same inspection tick delivers at most one bounded, fact-only
**coordination reminder** to the leader mailbox: run identity, pending/active
counts, pending node owners and the last durable progress time. Treat it as a
request to diagnose the run yourself — re-check dependencies, claim the next
task, verify model-turn boundaries, CI/rebase state or record an explicit wait.
Scanning (30s by default) and delivery are independent: a queued reminder
suppresses later ticks, and one stalled-progress epoch carries at most one
reminder plus one follow-up. The follow-up appears only after the leader
runtime recorded, on that exact message's ACK, that the turn carrying it ended
and a bounded cooldown elapsed; an ACK alone proves nothing about the turn, so
without that fact the phase stays unknown and nothing is re-sent. If the run
stalls again after real progress, the new progress epoch gets its own single
reminder. The reminder never resumes,
retries, claims, stops, starts a run, changes a generation, edits the Chart or
resets a continuation budget; explicit pause/cancel/operator stop always wins.
Closing the Workbench page does not stop the tick or the reminders — the page
only reads the facts the server already recorded.
There is no separate text-to-chart IR or parser.
If the request is underspecified or the generated definition fails `validate` or
`plan`, keep the definition out of `run` and report the missing decision to the
user. On recovery, read the authoritative file and original `runId`; never regenerate
the Chart from the natural-language request. Preserve per-attempt revision evidence.

- Read [cmd-reveal.md](references/cmd-reveal.md) for relevant discovery commands.
- Before creating or operating workers, read
  [team-execution.md](references/team-execution.md) for lifecycle, readiness and
  recovery boundaries, including current launch-binding limitations.
- To configure another agent system on this substrate, read
  [agent-system-deployment.md](references/agent-system-deployment.md): runtime
  paths, shared roots, launch bindings, public monitor API and preflight.
- For parallel work, verification gates or bounded repair, use
  [complex-workflows.md](references/complex-workflows.md) and its complete JSON
  templates. Substitute real task inputs; templates are not admitted runs.
- For a multi-issue feature with an integration branch, use
  [checkpoint-adapters.md](references/checkpoint-adapters.md). It explains the
  outer checkpoint DAG / inner agent-team pattern, acceptance and shared progress;
  the business adapter is task-owned, not a hidden built-in feature API.

This skill owns orchestration, not the members' work methodology. Obtain task
steps, role instructions, acceptance criteria and iteration limits from the user,
the referenced plan or applicable project skills. Do not add TDD, review stages,
test/build commands or staging steps simply because a team is being created.

Preserve requested roles and CLI choices. A default Codely controller uses the
current session without adding model/auth overrides. Use a separate execution
member only when the requested work needs one; do not assume this interactive
controller is already a persistent worker. Execution members default to
YOLO-equivalent tool approval using each CLI's own arguments, as specified in
the team execution reference. This does not change the interactive controller's
approval policy or authorize extra Git changes, publication or deployment.

## Reveal before authoring

Use the tools and schemas already advertised in the current session. In a
native Codely chat, `swarm_manage` action `status` shows whether the session
already owns a team. For a requested new team, use `create` in that session.
Look up a specific subcommand's `--help` when its arguments are unknown.
[Command discovery reference](references/cmd-reveal.md).

```powershell
codely --help
codely --cmd "/model"                          # models for current auth
codely --list-tools
codely skills list
codely mcp list
codely swarm workflow --help
codely swarm workflow schema --json            # only allowed IR keys
```

Re-run `schema --json` whenever validate rejects fields (launch/spec drifts).
Default model discovery is `codely --cmd "/model"`, without `--auth`.
Only use an auth override when explicitly requested by the user. Leave default
models unspecified; put explicitly selected models and effort in launch args,
not a second runtime configuration. Verify selected model ids against discovery;
do not silently replace a missing model. A model list alone does not prove effort
support. Discover external CLI flags through that CLI's own help.
Feature flags (`--experimental-*`, `--skills`, `--exclude-tools`, …) change what
is visible — re-list after toggling.

## Chart discovery (when a Chart is requested)

```text
schema → capabilities → examples/init → validate → plan → run
```

1. `codely swarm workflow schema --json`
2. `codely swarm capabilities --team <team> --json` (roster + static catalog;
   `probes` false ≠ live model list — use `--cmd "/model"` for that)
3. `examples --json` / optional `init --template … --output …`
4. Author JSON using **schema keys only** (including `launch` if listed)
5. `validate --file … --json` → must `ok: true`
6. `plan --team … --file … --json` → `runnable: true`; follow the team execution
   reference to resolve roster gaps and verify actual worker readiness
7. `run --team … --file … --run-id <uuid-v4>`
8. `status` / pause / resume / cancel / retry as needed

`codely swarm list --json` returns `items` and `nextCursor`. For another
page, pass that cursor with `--cursor`; a null cursor ends the list. Omit
`--json` for the text table and leader attach hint. The Workflow discovery commands above support `--json`; use their
`WORKFLOW_*` errors to correct the Chart before admission.

## Save and reuse a Chart template

The Chart itself is the template metadata: `definitionId` is the catalog
`name`, `title` is the display title, and `description` states when to use it.
Never keep a second manifest of names or descriptions, and never write those
values into member `description` (that field is a member's business role).

When the user asks for a reusable workflow, or asks to save one:

1. Collect a concrete name and purpose and write them into the Chart
   (`definitionId`, `title`, `description`). A description is persisted, not
   only spoken in the conversation.
2. Validate the Chart first, then save the managed, self-contained copy:
   `codely swarm template save --chart-file ./workflow.json`
   (`--name` / `--description` override the managed copy only; `--force`
   replaces an existing template; a name conflict fails by default).
3. Discovered templates come from real files:
   `codely swarm template list --json` →
   `{name, description, chartFile}`; `codely swarm template show <name> --json`
   reads the complete Chart and its packaged resources. Neither command starts
   a worker or admits a run.
4. Choose a template by its `description`; when the user names one, `show` it
   before use. If nothing matches, use the normal authoring flow instead.
5. When the run starts from a fresh team in your current Codely or ACP
   session, create and attach it in place with the `swarm_manage` tool:
   `{"action": "create", "team": "<name>", "member": "<member>",
"description": "<purpose>"}`. The current session becomes the leader, so
   its own lifecycle consumes the admitted task, and the purpose is persisted
   as durable team metadata. Never run `codely swarm create` to create a team
   that your current session should own: that command starts a separate
   interactive member session, and a non-interactive shell child exits
   immediately, leaving a stopped leader your session was never attached to.
   Use `codely swarm create --team <name> --member <member> --description "<purpose>"`
   only when the user explicitly wants to launch a standalone interactive
   member session in a real terminal and keep that process running. After
   exiting an interactive leader terminal, return to the original leader chat
   with `codely swarm attach --team <team>` (using only the team name): it
   restores the original leader identity, mailbox and canonical JSONL history
   instead of running `worker resume` (which starts a background headless
   worker) or `--resume-session` (which only loads ordinary history without
   Swarm leader identity).
   Team purpose is persisted metadata, separate from a Chart template
   description and from any member's business role.
6. Prepare members with the launch the run itself will execute, whenever the
   Chart pins a `launch`, `model`, `reasoningEffort` or external executor:
   `codely swarm template instantiate <name> --json` (same optional `--inputs`)
   returns the exact definition `template run` admits. Its
   `definition.members[].launch` is the bound resource identity: package
   entries point into the template store's immutable content tree, not the
   replaceable `<templateRoot>/<name>` directory. Pass exactly those bindings
   to `codely swarm worker prepare --team <team> --members <file.json>`
   (`[{"name": "<member id>", "launch": <that launch>}]`). Admission keeps the
   prepared launch unchanged, so prepare, `workflow plan`, the durable task
   snapshot and the first claim all compare one identity. A member prepared
   from a path rebuilt out of the managed package (or from `show`'s relative
   entries) is a real config mismatch: `template run` fails closed as a
   conflicting execution binding before any task is written.
7. Instantiate into a **new** task and run:
   `codely swarm template run <name> --team <team> --run-id <uuid-v4>`
   with optional `--inputs <file.json>` = `{ "<nodeId>": <input> }`. Bindings
   are explicit node inputs; there is no template language and no global text
   substitution. Relative launch/script entries resolve against the template
   package, so the caller's working directory does not matter.
   The binding reaches the claimed task's model-visible `inputs`: an object
   binding contributes its own keys (`inputs.ticket`), any other JSON value is
   the single input named `input` (`inputs.input`), and a name the binding
   defines wins over a same-named `inputsFrom` projection, which fills only the
   remaining names. A `schemaVersion` 1 Chart cannot carry node input at all:
   both `template run --inputs` and a Chart-authored `nodes[].input` fail
   closed instead of being silently dropped, so use a `schemaVersion` 2 Chart
   when input is required.
8. `save` is not `run`, and instantiating a template is not resuming a run:
   a new task always gets a new `runId`; recovering an earlier execution keeps
   its original `runId` and admission. The template package is never rewritten
   by instantiation, and old run state/PASS is never inherited.

## Reuse a ready Chart or Swarm

The CLI can recover a configured team without a Control-plane. When any required
monitoring condition above holds, restore and verify Control-plane before
admitting new work or resuming execution; CLI availability does not waive that
requirement.
Read the saved inputs first — never guess them from the old conversation and
never edit durable records:

- the authoritative `WorkflowDefinition` JSON (the editable source),
- the actual `launch`/`executor`/`env`/`root` values used at prepare time
  (the prepare `--members` JSON when the Chart does not embed a full launch).

Then follow this order:

1. `codely swarm list` / `codely swarm status --team <team>` locate the team and
   its current members. Git root/`CODELY_SWARM_ROOT` must be the same root the
   first round used.
2. `codely swarm workflow list --team <team> --json` finds earlier runs of this
   Chart; open one with `codely swarm workflow status --team <team> --run-id
<runId>`. One Chart may have several runs — never pick "the" run by
   `definitionHash` alone.
3. Check the member's execution mode and readiness. For persistent workers,
   reuse a healthy matching launch, resume eligible `stopped`/`stale`
   members, or prepare missing members. For on-demand members, follow
   [headless-members.md](references/headless-members.md). `plan.runnable`
   checks the roster and launch bindings.
4. To change a launch/model/executor, stop the old work first and use the
   existing release → prepare/spawn order; the old binding is never
   overwritten in place. An old run that must be kept is not cancelled
   automatically.
5. Re-running the same or an edited definition always uses a **new runId**
   (`validate` → `plan` → `run`); only recovery of the same execution reuses the
   original `runId` and its admission (`resume`/`retry`). Session continuity
   follows the execution mode. Persistent workers can reuse their session,
   local headless reuses its canonical session across activations, and remote
   on-demand opens a new ACP session per assignment.
6. Report the source file, `definitionHash`, `runId`, and the actual
   claim/result. A written file, a `ready` member or an admission is not
   completion.

## Hard rules

- Same format as hand-written charts; no alternate IR/scheduler.
- Never `run` before successful `validate` and `plan.runnable: true`. Neither
  `runnable` nor `attached` alone proves a persistent worker is ready. If the
  installed launch path cannot satisfy this order, report the concrete missing
  wiring; do not bypass it with dummy tasks, durable-file edits or fake members.
- Persist the JSON; resume that file — do not regenerate on recovery.
- Apply the explicit Control-plane decision above before admission. If required
  coverage is unavailable, keep new work out of admission, report the gap and
  restore observation; do not invent an HTTP scheduling endpoint.
- Team/worker tools need `--experimental-swarm` (or `swarm create|join|attach|worker`);
  one-shot `codely swarm workflow …` does not.
- Prefer `workflow` over deprecated `chart` alias.

## Translate task dependencies and outcomes

Keep the user's requested roles distinct. The controller coordinates assignments
and handoffs; member responsibilities and completion criteria come from the task.
Discover supported schema fields before translating a conditional plan.

- Use `dependsOn` for ordering, `inputsFrom` for committed predecessor data and
  `resultSchema` for the task's structured outcome. Execution success and business
  outcome are distinct: a successfully completed assessment may request more work.
- Use `when` against structured results for branches, not prose keyword matching.
  Expand a user-specified iteration bound into uniquely named nodes; do not invent
  a fixed number of rounds. Map requested early-exit and human-decision conditions
  to the supported skip/pause behavior.
- Do not create an ordinary all-dependencies join across mutually exclusive
  branches. Follow the task's completion criteria for each selected branch.
- Use `command` nodes for deterministic operations specified by the task. Their
  actual execution outcome must be recorded; the task defines the meaning of
  success. Do not replace a command result with a model's assertion.
- A successful command returns `{exitCode, stdout, stderr}` with `exitCode: 0`;
  a nonzero exit, signal, timeout or cancellation fails the node instead of
  recording that envelope. JSON printed to stdout remains
  a string. `inputsFrom` is not automatic command argument/environment binding.
  Pass the task's frozen input file explicitly and choose `command.timeoutMs`;
  omitting it currently inherits a 30-second process deadline, not infinity.
- An omitted or unset member turn timeout is unlimited.
- Only an explicit positive integer narrows the member-turn budget.
- Command, preScript and tool timeouts are independent.
- Reconcile uncertain effects before retry. For a wedged unlimited turn, check
  whether pause/cancel requested active-turn control, or verify exit through
  the worker stop operation. Set `launch.turnTimeoutMs` for a finite budget.
- Store model/auth/effort only in launch args/env. An external CLI uses the
  schema's `external-cli` executor and its verified entry/args. Do not append
  Codely membership flags to the external CLI.
- On recovery, inspect durable results before redelivery. Reuse the original
  CLI session where supported and report missing recovery wiring.
- Remote join credentials remain gateway-owned. Follow the execution reference
  for bootstrap-bound retry, HTTPS deployment and Control-plane's observation
  boundary; never repair credentials by editing member records.

If the user requests one Chart, expand phase dependencies and bounded repair
rounds into that one persisted definition; do not silently split it into runs.
If a limit prevents this, report the actual limit. For natural-language execution
evidence, retain the request, generated definition and durable run outcome.
Generated JSON is not completion; repeated LLM generations need not have identical
hashes, but recovery must reuse the frozen definition and run identity.
