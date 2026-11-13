---
name: swarm-workflow
description: >-
  Organize Codely agent teams and optional Swarm WorkflowDefinition runs from
  natural-language goals, roles and constraints. Use for teammate collaboration,
  role-based collaboration, durable charts, editable per-team DAG HTML,
  live Chart file editing, multi-checkpoint delivery, or agent
  systems built on Swarm and Control-plane.
  Prefer this over inventing a parallel planner: explore Codely cmdarg/--cmd and
  `codely swarm workflow` as ground truth (N1 = same format + same run path).
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
For ordinary team collaboration use existing membership, worker, mailbox and
shared-task tools without requiring a Chart. For ordered work or bounded repair
loops, author a `WorkflowDefinition` and use the same durable execution path as
a hand-written chart. **Never guess** model ids, auth, tools or chart fields.

## Choose the member execution mode

Existing `swarm worker` members retain their persistent execution behavior.
Use the separate `swarm headless` surface only when the user requests explicit
on-demand execution: register a member without a process, activate one ready
work unit, then let that instance finish and exit normally. The member and its
canonical session remain available for the next explicit activation.
Discover `codely swarm headless --help` first and read
[headless-members.md](references/headless-members.md). Do not simulate this mode
with a persistent worker plus a normal-success `stop`, or convert an existing
member. Headless is an execution lifecycle, not a model/executor selection.

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

Explicit, attended headless activations are also allowed without Control-plane:
the caller waits for each activation's terminal outcome and explicitly starts
the next one. Restoring the same completed member/session for a second such
activation does not by itself require unattended recovery. An inactive member
does not wake itself. The current headless surface does not implement
Control-plane automatic DAG activation; a running monitor is not evidence that
headless successors will start. Do not promise unattended headless execution.

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
nextCursor and expanding canonical references when needed:

    codely swarm events --team <team> --member-id <targetMemberId> --limit 50 --json
    codely swarm events --team <team> --member-id <targetMemberId> --after <journalId:cursorSequence> --limit 50 --expand --json

The mailbox sequence printed by `swarm_send` is not the journal cursor sequence.
Match evidence references to both the mailbox message ID and its sequence; check
the receiver member/tenure/generation on the committed and ACK facts. Read the
canonical recipient conversation record when the ACK reference needs
verification. An ACK proves the message was committed to the recipient's
canonical session. If that same ACK carries turnEndedAt, the model turn carrying
the steer also ended. Neither fact proves the requested task completed.

Treat a matching journal fact as positive evidence. A missing fact is
inconclusive: the canonical conversation or ACK write can succeed while its
subsequent journal fact is absent. In that case, use the installed version's
read-only exact transcript/ACK lookup. If no such lookup is exposed, report
consumption as unknown and request a correlated task or result receipt; do not
infer failure from a missing journal row or an empty mailbox.

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
If only accepted or committed is proven, report that phase and leave consumption
pending or unknown until exact recipient evidence is available.
An empty inbox, an idle member or a local coordinator log does not prove
consumption. Do not resend the steer or create a duplicate task until its
possible existing side effects have been checked.

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

Do not guess. Explore cmdarg / `--cmd` / subcommand `--help` first (full map:
[cmd-reveal.md](references/cmd-reveal.md)).

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

Discovery JSON commands need `--json`. Fix via `WORKFLOW_*` envelopes — do not
bypass validate/plan.

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
   member session in a real terminal and keep that process running.
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
3. Judge members, not just `plan.runnable`: a healthy member with the same
   launch is reused; `stopped`/`stale` is resumed; a missing member is
   spawned/prepared. `plan.runnable` is a roster check, not worker readiness.
4. To change a launch/model/executor, stop the old work first and use the
   existing release → prepare/spawn order; the old binding is never
   overwritten in place. An old run that must be kept is not cancelled
   automatically.
5. Re-running the same or an edited definition always uses a **new runId**
   (`validate` → `plan` → `run`); only recovery of the same execution reuses the
   original `runId` and its admission (`resume`/`retry`). A new runId is not a
   new worker conversation: reused/resumed members continue their session.
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
- Team/worker tools need `--experimental-swarm` (or `swarm create|join|worker`);
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
