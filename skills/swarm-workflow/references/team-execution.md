# Team execution and recovery

Use the installed CLI/tool contracts. This reference describes execution
invariants, not new TeamSpec/apply commands. Do not invent an unsupported API.

For a complete host-side setup (runtime paths, shared root, launch JSON and
monitor routes), read [agent-system-deployment.md](agent-system-deployment.md).
For outer checkpoint commands that own inner agent teams, read
[checkpoint-adapters.md](checkpoint-adapters.md); that composition uses the same
runtime, but its business verification and Git integration are task-owned.

See [headless-members.md](headless-members.md) for local explicit activation
and the attached remote on-demand host. The preparation and resume steps below
describe persistent workers. Local headless uses `headless activate` to start
each unit and exits on normal completion.

## Choose the execution shape

- Flexible collaboration uses team membership, messages and shared tasks
  through `swarm_manage`, `swarm_worker`, `swarm_send` and `swarm_task`.
  The shared task board belongs to the team and `job_*` tasks belong to a session.
- A Workflow uses a Chart to declare dependencies, conditions, structured
  result handoff or bounded repair rounds. Reviewer is a task responsibility.
  Membership permissions follow the assigned Swarm role.
- A remote on-demand host runs Gateway-ready Chart nodes bound to its joined
  member. Mailbox work requires a recipient that consumes it. Keep ordinary
  shared tasks in a separate team or use persistent workers for a mixed backlog
  because unbound ordinary tasks can be offered to the on-demand host.
- Example ("have `reader` read README, then have `reviewer` verify it"):

| User intent                                                       | Dispatch path                                                                                | Receipt & completion signal                                                                                                                                                           |
| ----------------------------------------------------------------- | -------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Ad-hoc collaboration with two named recipients | `swarm_send` with persistent recipients or explicit local headless activation. Send `work` to `reader`, who hands off to `reviewer`. | The send receipt confirms mailbox acceptance. Confirm completion from the reviewer's `result` reply to Leader. |
| Team backlog showing who claimed each step and what is blocked | `swarm_task` with persistent workers. Create two tasks with `blocked_by` and agree where to read the summary. | Check owner, blockers and completion by `task_id` with `swarm_task status` or `codely swarm task status`. `blocked_by` gates claiming. |
| Repeatable two-step flow with automatic structured output handoff | Chart (`dependsOn` + `inputsFrom`)                                                           | Receipt returns `runId` and `codely swarm workflow status` lookup; inspect both nodes' committed results and same-run `inputsFrom` lineage by `runId`.                                |

- Preserve explicit model/executor choices. The controller remains on the
  default Codely model unless requested otherwise. A role file affects the worker
  only through a supported CLI/env entry; a `skillRef` label alone is not proof
  that its instructions were loaded.

## Prepare and start persistent workers

1. Read current team/roster and relevant workflow/worker help. Reuse only a team
   belonging to this work. Record the intended workspace and member names.
2. Persist the actual launch configuration before using it: `entry`, `cwd`,
   `env`, optional ordered `preScripts`, and `args`. Scripts use an explicit
   interpreter. An omitted or unset member turn timeout is unlimited.
   Only an explicit positive integer narrows the member-turn budget.
   Command, preScript and tool timeouts are independent.
   Reconcile uncertain effects before retry.
   For a wedged unlimited turn, check whether pause/cancel requested control
   of the active turn, or verify exit through the worker stop operation. Set
   `launch.turnTimeoutMs` for a finite budget.
   Apply the worker approval contract below. Do not inject provider
   configuration per node, auto-add `--auth`, or mutate shared `/model use` settings.
3. `swarm create` / `swarm join` / `swarm attach` attach the current CLI
   interactive terminal session (`codely swarm attach --team <team>` re-enters
   an existing team as its original leader). They are not
   one-shot provisioning commands to timeout and abandon. `worker spawn`
   creates membership itself; do not join a member and then spawn that same
   attached name. Resume stopped/stale background workers through the existing
   controller (`worker resume` runs in background headless mode);
   do not duplicate healthy members or assume a stale PID belongs to this work.
4. Verify the worker controller's readiness result and current fence/presence,
   not just a roster row, a PID or `plan.runnable`. The persistent worker runs
   through the launcher's internal Swarm fence. Never construct that fence by
   hand. A separate `codely -p` success is not a worker readiness test.
5. For a Chart, inspect `validate` and `plan` envelopes, then admit using the
   frozen definition/run ID. Observe a real claim/result; admission is not
   execution. For a non-Chart team, deliver authorized work through the existing
   mailbox/task tools after readiness.

### Durable launch binding

Discover `codely swarm worker prepare --help` and `worker spawn --help` on the
installed version. The current source exposes `prepare --team ... --members ...`
for a JSON array of name/launch/description, and `spawn --launch ...` for an
individual member. These persist a member binding before spawn; a chart file on
disk alone is not that binding. Existing Chart task snapshots remain frozen.
External executors also need the supported carrier binding; do not mistake an
`agent` entry alone for an external executor declaration. Verify actual readiness
through the installed path. Older versions with Chart-only binding may not
satisfy prepare-before-plan; report that gap without dummy tasks or durable edits.

## Stop, release and resume a managed worker

`codely swarm worker stop` stops the process; it does not retire the tenure.
The default preserves member, tenure and canonical session so the same worker
can be resumed with its conversation. Repeating a default stop is idempotent
and never turns into an implicit leave.

- `--release` is the only way to free the name and end the tenure. It happens
  only after the process exit is confirmed; an unconfirmed exit reports the
  reason and leaves the tenure intact for a retry.
- A parked record is not an exit observation. `stopMember` releases the owner
  pid in the same write that parks the tenure, so `state: stopped` with
  `pid: null` proves nothing about the OS process (a worker may still be
  flushing its transcript). The born identity of the generation
  (`processIdentity`: pid / birth / host / boot) and any ownership captured on
  the stop operation survive the park. A later, independent CLI can continue
  the stop when it can bind that same-fence durable evidence on the executing
  host. Missing or conflicting identity, incomplete descendant coverage, or an
  unconfirmed process exit keeps the operation `unconfirmed` and blocks
  `--release`.
- Durable evidence is bound to one executing host and one boot. A stop whose
  recorded root identity contradicts the member's born identity, names a
  different pid, or was captured on another host/boot reports `unknown` with
  the binding reason and sends no signal — including no escalation of a child
  this process still owns. Never edit the durable JSON, never trust the
  newer-looking token, and never kill a bare pid to make the record agree;
  retry `worker stop --operation <id>` on the executing host instead.
- Every stop writes a durable operation first (fenced to member / tenure /
  generation / instance, with the control request id it will use). A different
  CLI can query it with `worker status --json` (per-member `stop` block) and
  retry exactly that operation with `worker stop --operation <id>` — the
  operation pins its own target, so no `--member`/`--all-workers` is needed or
  allowed; the retry reuses the same request id instead of issuing a second
  stop. A retry only ever adds evidence: the original root birth is never
  replaced and a captured descendant is never dropped.
- Exit is verified for the root **and** its managed descendants. The tree is
  captured (pid + birth token) while the root is still visible — Linux `/proc`,
  Darwin `ps`, Windows CIM creation time; a dead or reused root therefore never
  proves the children it owned are gone. The capture also records how much of
  the tree it covered: `complete` (every descendant was enumerated), `root_only`
  (the root was identified, children were not), or `unknown` (legacy records).
  An empty descendant list from `root_only` or `unknown` never means "no
  children". If the original root is still verifiable, a later local CLI can
  capture its tree and enrich the same operation. Until coverage is complete
  and exit is confirmed, the operation cannot become `exited` or be released.
  If one captured descendant is still alive, the root and the descendants are
  judged separately: the root can read `confirmed` while the operation stays
  `unconfirmed`. A stop never re-discovers that child through the current pid
  table. An explicit `--force` settles exactly the captured descendants: every
  target is validated against one fresh table read before any signal, so an
  unreadable birth aborts with "no signal was sent"; every signal is preceded
  by its OWN birth-token re-read (POSIX table read, Windows per-handle creation
  time), including every SIGKILL escalation, so a pid that was reused after the
  capture — or after an earlier signal in the same batch — is never signalled.
  If the table
  becomes unreadable (or a signal fails) only after some targets were already
  signalled, the error names the signalled and unresolved pids instead of
  claiming "no signal was sent", and the same operation is persisted as
  `unconfirmed` with those per-target outcomes. On Windows this includes a
  handle batch that settles only part of the captured tree (a descendant
  terminated while the root is still `pending`/`unverifiable`): every target's
  outcome is recorded, so the next CLI continues the same operation instead of
  guessing. A force that still cannot
  confirm the captured children leaves the operation `unconfirmed` with the
  captured birth evidence preserved; continue it with
  `worker stop --operation <id>` on the executing host rather than editing the
  durable JSON or killing pids by hand.
- `--force` requests verified termination on the executing host. The root's
  worker identity is checked before entering the force path. Before each signal,
  the platform-specific terminator verifies the target's captured
  birth token or process-handle identity. The live-root path terminates the
  managed tree; the root-gone path acts on the captured descendants. An unknown
  or changed identity is never used as a signal target. Windows binds each
  captured target to its own process handle, matches that handle's creation
  time to the captured birth token, and confirms exit on the same handle. A
  recycled pid is refused and an unreadable handle keeps the operation
  `unconfirmed`. A remote-host pid is never probed or signalled as a local pid.
- Team scope is explicit: `--all-workers` selects managed execution members only
  (never the leader, never an interactive join); `--only-idle` additionally
  requires no outstanding tasks or unacked messages across the member's runs —
  every candidate is checked, including a `stopped` member with no pid, and an
  unreadable or truncated scan is never treated as idle; `--dry-run` reports
  the selection without writing or signalling anything. `--json` prints the
  same facts the human line summarizes. Every selected target whose exit is
  unconfirmed counts as a failure of the `--all-workers` run: a parked or
  unconfirmed target is never removed from the denominator.
- `--deadline` is one budget for one cooperative attempt. Omitted, non-numeric
  or non-finite values use 30000 ms; valid values are clamped to 1..300000 ms.
  The deadline bounds the wait, never the durable operation: a timeout reports
  `unconfirmed` and leaves the same operation for a later CLI to continue.
- Inspection never starts or resumes a worker. An unresolved stop intent marks
  that exact fence as operator-stopped; an explicit `worker resume` runs in
  background headless mode, is the atomic end of an operator/idle stop intent
  and mints exactly one new generation; a late retry of the old request never
  affects the new generation. A resume keeps the canonical session and does not
  replay completed tasks. A resume is not evidence that the old orphan exited:
  it only ends the stop intent for the old fence, so the old process tree must
  be settled by its own operation before anyone claims it is gone.
- `worker resume --readiness-timeout <ms>` (tool parameter `readiness_timeout`)
  sets the readiness wait budget (default 60000 ms, valid integer 1..600000 ms;
  invalid values fail closed with `SWARM_INVALID_NAME`). The fixed native
  Swarm tool under `claude_code_compact` uses its built-in readiness behavior.
  Readiness failures are classified distinctly:
  - `SWARM_RESUME_TIMEOUT`: the child did not publish same-fence presence
    within the budget; output includes the actual budget, `launch=...ms,
wait=...ms` timing split, bounded stderr, and — when process exit could not
    be confirmed — the pending `startup_cleanup` operation id (`op=<id>`).
  - Early process exit: reports `process exited (code=..., signal=...)` (not
    `SWARM_RESUME_TIMEOUT`) with bounded stderr.
  - `SWARM_RESUME_NOT_ALLOWED`: the member's state does not allow resume (e.g.
    `attached`, `left`, `headless`), or a prior failed start left an unresolved
    `startup_cleanup` operation. Confirm the old process tree exited with
    `codely swarm worker stop --team <team> --operation <id> --force` before resuming again
    so two live workers never share the same workspace. If startup cleanup
    lost the process tree before it could bind its descendants, a missing root
    PID alone leaves resume and release blocked. `worker status --json` shows
    the operation and its exit evidence.
- Resume only promises the restore scope a backend actually verified. The
  native Codely session and the external carriers' structured session ids
  (`session-mismatch` / `missing-session-id` fail closed) are covered; a
  backend that cannot restore must fail explicitly and never silently starts a
  fresh session and calls that a resume.
- A remote-admitted worker's stop is executed by its own host over the gateway:
  the authority records the exact-fence operation and never probes or signals the
  remote pid (it could not interpret it). The host polls
  `GET /v1/swarm/stops/pending` with its member token, verifies the local tree
  itself (pid + birth token), stops it, and reports the birth-fenced evidence
  back to the same operation. Until that evidence arrives the operation stays
  `unconfirmed`; a host that cannot name the exact process reports `unknown`
  rather than a pid-only claim. An old token/fence is rejected, so a late stop
  for one generation never acts on a newer one. Only the executing host reports
  evidence; the authority never probes or signals a remote pid.
- The host persists its capture BEFORE it signals anything:
  `POST /v1/swarm/stops/<id>/capture` stores the born identities (root and
  every captured descendant, with its coverage) on the same durable operation,
  and the response is the durable ACK. If that write fails the host stops
  nothing and retries the same operation later — an untracked stop could orphan
  children nobody can attribute. `stops/pending` returns the stored capture, so
  an observer that starts (or restarts) after the root already exited continues
  from those identities instead of re-reading a reparented tree.
- A stop is closed only by a durable terminal ACK: root **and** managed
  descendants confirmed, persisted on the authority. Until then the executing
  host keeps only its evidence observer alive (execution and heartbeat stopped,
  no new assignments) and re-checks the same `operationId`/`requestId` on later
  ticks; a dropped POST, a lost response or one timed wait is retried, not
  turned into a success or into a new operation. Each attempt is single-flight
  and bounded by its own budget, so a worker that never answers the close
  cannot wedge the observer. Explicit shutdown, a revoked token or a superseded
  generation still releases the observer, but never reports a fake exit.
- A parked (`stopped`) member keeps exactly one management capability: its own
  unresolved stop operation. It may read `stops/pending` and submit capture or
  evidence for that operation — nothing else. Any operation id, generation,
  tenure or instance mismatch, a left/revoked tenure or an already settled
  operation stays rejected, and the stop-evidence grant never restores work
  authority (no heartbeat, claim or assignment).
- A POSIX `--force` walks the live tree while the root is still present; once
  the root has already exited, the captured descendants are settled one by one,
  each pid re-read and birth-matched immediately before its own signal. Windows
  settles the same captured set through one handle per target. Both paths need a
  `complete` capture: with `root_only`/`unknown` coverage no platform can
  enumerate the reparented children, so the operation stays `unconfirmed`
  instead of guessing. Never infer "no children" from a table read taken after
  the root exited.
- Platform limits: Linux has a kernel boot identity and `/proc` start time;
  Windows and Darwin currently report no boot identity, so the born token and
  host id are the binding evidence there. A conflict that cannot be resolved
  from durable evidence stays `unconfirmed` with its reason instead of being
  guessed from hostname, timing or a bare pid, and a missing or malformed
  process-table read is never treated as "the process is gone".
- Compatibility: older builds implemented a second `worker stop` as a
  leader-side release. Treat `stop` on those versions as `stop --release`, and
  verify the exit yourself before relying on the name being reusable.

## Re-run, edit and recover a configured team

A team using persistent workers that already ran once is reusable from public commands alone. This
technical capability does not waive
[the Control-plane decision](../SKILL.md#decide-whether-control-plane-is-required).
When coverage is required, restore it before new admission or resumption.
Start from the saved inputs: the source Chart file and
the prepare `--members` JSON that carried the actual `launch`/`executor`.
Do not reconstruct parameters from the old conversation and do not edit durable
member/task/run records to make a flow work. The source definition is an
editable input; an admitted run is frozen history.

1. `codely swarm list` and `codely swarm status --team <team>` locate the team.
   For a stuck diagnosis, read the shared action ledger next:
   `codely swarm events --team <team> --limit 50 --json`. Add `--expand` only
   when canonical task/result/message evidence is required. The same ledger is
   available to an attached agent with `swarm_task` action `events`; omit
   `task_id` for the team journal, or include it for the legacy per-task
   workflow-event tail. An empty incremental page is not proof of stuck work.
   The same `CODELY_SWARM_ROOT` (or the same Codely home) must be in use; a
   different root is a different set of teams and runs.
2. `codely swarm workflow list --team <team> --json` enumerates the Chart runs
   of that team; `codely swarm workflow status --team <team> --run-id <runId>`
   reads one. The same Chart may have several runs — `definitionHash` identifies
   a definition, not "the" execution.
3. Prepare judges each member by its refreshed presence: a healthy member with
   the same launch is reused, a `stopped`/`stale` member is resumed, a missing
   member is spawned. `plan.runnable` only says the roster covers the Chart; it
   does not prove a persistent worker is ready.
4. A changed `launch`/model/executor is never written over an existing binding:
   `prepare` rejects the conflict and names the member. Check the old run, then
   do the stop/release for that tenure:
   `worker stop --member <name> --force --release` verifies the root and its
   managed descendants before freeing the name. After that, run
   `prepare`/`spawn` with the new launch. A parked record (`pid: null`) can still
   be stopped when its same-generation durable identity can be verified on the
   executing host. Missing or conflicting identity, incomplete descendant coverage,
   or an unconfirmed process exit keeps the operation `unconfirmed` and blocks
   `--release`; continue the same stop operation when the missing evidence is
   available. Use `resume` only to continue work with the existing launch: it
   neither frees the name nor proves that the old process tree exited.
   Old open runs are not cancelled automatically; keep or cancel them
   explicitly. Do not pass a changed launch to `resume`.
5. Re-running the same or an edited definition uses a **new runId** through
   `validate` → `plan` → `run`: new tasks, same `definitionHash` only when the
   definition is unchanged. Only recovering the same execution reuses the
   original `runId` and its admission (`resume`/`retry`). A new runId is not a
   new worker conversation; reused or resumed members continue their session.
6. Report the source file, `definitionHash`, `runId`, and the real claim/result.
   A written file, a prepared member, `ready` or an admitted run is not
   execution evidence. Already committed results are never replayed.

A failure summary gives the first failed member and its main cause plus the next
command; remaining failures are only counted. Treat that as the diagnosis, not
as permission to bypass the binding or lifecycle checks.

### Control-plane requirement

Use Control-plane for requested monitoring/maintenance, work outliving its
initiating controller, coordination across teams/runs, or unattended recovery
and unattended repeated/restartable operation. Reuse one persistent instance for the same
root; do not start a separate monitor for every team. The bounded single-team,
continuously attached exception is defined in the skill's decision checklist.
`codely serve control-plane --help` describes the installed surface:

```text
codely serve control-plane --host 127.0.0.1 --port 4040 \
  --swarm-root <the same absolute root the CLI uses> --no-rollout
```

Keep it in its own terminal or an existing process manager, and reuse an
already running instance for the same root. Check the printed address,
`/__codely__/health`, and the target team under
`/__codely__/control-plane/swarm/teams`. A healthy response does not prove the
root is the one the CLI uses — verify the team appears there. Swarm HTTP is
loopback-only by default: a bind outside the `127.0.0.1`/`::1`/`localhost`
allowlist answers `/swarm/...` with 403 unless the operator passes
`--allow-swarm-api`. That opt-in exposes leader-authority access with no
operator authentication, so use it only on a trusted network or behind an
authenticating proxy. Without it, an attached leader still observes durable
notifications; when both are gone there is no live observer, but durable state
remains readable afterwards. Those fallback capabilities do not satisfy a
required monitoring condition. There is no template edit/prepare/rerun HTTP
endpoint; do not invent a shortcut. If the monitor is unavailable, use the CLI
to diagnose and reconcile existing work, but do not admit new monitored work
until coverage is restored.

Use [the preflight and outage procedure](agent-system-deployment.md#required-control-plane-preflight)
to verify the team before admission and the run afterwards. Report the actual
monitor URL/root/team/run identities, not just that port 4040 responds.

## Worker approval contract

All execution members default to YOLO-equivalent unattended tool approval,
including third-party CLI workers. Use the selected CLI's native arguments:

| CLI         | Native worker arguments                      |
| ----------- | -------------------------------------------- |
| Codely      | `--approval-mode yolo`                       |
| Cursor      | `--yolo` (alias `--force`)                   |
| Codex       | `--dangerously-bypass-approvals-and-sandbox` |
| Claude Code | `--dangerously-skip-permissions`             |

For a Codely interactive or ACP initiator session, `-y` is the short YOLO
spelling. Never pass `-y` together with `--approval-mode`; the CLI rejects
that combination (`Cannot use both --yolo (-y) and --approval-mode together`).
Codely persistent `swarm worker` launch records persist `--approval-mode yolo`
only: the worker launcher normalizes any `-y` / `--yolo` spelling to
`--approval-mode yolo` before spawn so a frozen launch cannot collide with the
forced mode.

Persist these native arguments in `launch.args` when preparing the team or Chart.
The Codely worker launcher also enforces its own approval mode; this only covers
the Codely process, including the carrier of an external worker. The external
child needs its own native approval arguments. Never inject Codely's
`--approval-mode`, `--experimental-swarm` or membership fence into that child.
Keep the persisted launch arguments on resume; do not add a separate approval
field or infer policy from model/provider or arbitrary wrapper names.

Verify flags against the installed CLI's help/version. Cursor's `--print` alone
is not YOLO; `--auto-review` may still ask. Codex `never` and Claude `dontAsk`
can deny operations instead of asking, so they are not full YOLO equivalents.
An explicit deny or enterprise restriction remains a visible failure; do not
retry it indefinitely. The native flags do not replace a business `needs_human`
gate and do not authorize extra Git or remote publication actions.

Cursor and Codex flags were checked through local CLI help. The 2026-10-19
acceptance also exercised real Cursor auto and Codex gpt-6-luna sessions with
skill/MCP context, peer messages and canonical result submission. That attached
worker evidence does not prove every managed carrier launch mode. Claude live
acceptance and rollout were explicitly skipped for this scope; preserve those
boundaries rather than reporting them as passed.

## Launch and communication semantics

- Resolve shared cwd/env once; ordered preScripts run shell-free before the final
  worker. A spawn error, timeout, cancellation, signal or nonzero exit stops the
  chain. stderr with exit 0 is not itself failure. Keep diagnostics bounded and
  redact resolved secrets. Only the final Codely carrier receives Swarm fencing.
- For an external CLI, distinguish Codely carrier readiness, external session
  initialization and committed task result. Use the existing driver with the
  selected CLI's verified arguments. `exit 0` or a session ID alone is not a
  successful assignment.
- `swarm_send` acceptance does not prove the recipient processed the message.
  Check the relevant recorder/result/ACK. Do not assume a stopped recipient is
  automatically restarted by sending, or treat idle as task completion.
- For a steer, retain the returned `messageId` and mailbox sequence together with
  the team, target member, tenure and generation. The send receipt proves durable
  mailbox acceptance only. Verify the matching `message_committed` and
  `message_acked` journal facts, following `nextCursor` for pagination and expanding
  their canonical message references as needed. Match both messageId and mailbox
  sequence to the `mailbox_message` reference; the journal cursor sequence is a
  separate sequence space. Check receiver fence identity and, when needed, the
  canonical conversation record referenced by the ACK.
- Matching journal facts are positive evidence; missing facts are inconclusive.
  A canonical transcript or ACK write may succeed while its subsequent journal
  fact is absent. Use the installed version's read-only exact transcript/ACK
  lookup when that happens. If none is available, report consumption as unknown
  and request a correlated task/result receipt; do not infer failure from an
  absent journal row or empty mailbox.
- A durable ACK proves the message reached that receiver's canonical session.
  If that exact ACK has `turnEndedAt`, the model turn carrying the steer ended.
  Neither proves requested work is done. For actionable steers, ask for a result
  correlated to the original `messageId`/sequence: received with matching ACK
  evidence; in_progress with task/run ID and owner; blocked with its actual
  prerequisite/error; or completed with a canonical result/artifact reference.
  These are requested reply labels, not automatic Swarm status values.
  Verify progress/results through `swarm_task`, workflow status or the durable
  result. Use the sender's supported return channel for the semantic report. A
  detached external CLI or outer chat is not implicitly woken by mailbox ACK.
  When only accepted/committed is proven, report that phase and keep consumption
  pending/unknown until exact recipient evidence is available. An empty inbox,
  idle member or local coordinator log does not prove consumption. Do not resend
  or duplicate a task until its possible side effects have been checked. If the
  sender is a team member, return a `kind=result` message to it; for an external
  sender, use its registered return channel.
- The existing persistent runtime polls tasks and has a bounded fallback wake.
  If work stays pending, inspect worker mode, liveness, claim ownership and
  dependencies first; do not invent a new queue or blame missing wake-up.
- Coordinate write scopes. Shared task ownership does not lock workspace files;
  serialize overlapping edits and heavy build/format operations.

## Recovery and evidence

### Terminal interactive leader re-entry (`codely swarm attach --team <team>`)

Terminal interactive re-entry uses `codely swarm attach --team <team>`, while
ACP clients recover a leader session through `session/load` with the canonical
binding or trusted `_meta.codely.swarmRecovery` locator. Both keep the same
leader member, tenure, mailbox and canonical JSONL transcript, and never fork a
second history authority. Do not substitute `--resume-session` (ordinary
history without Swarm leader attachment) or `worker resume` (background
headless mode) when the user wants to return to interactive leader chat.

- **Stopped or stale leader**: validates workspace containment and canonical
  transcript ownership, claims the OS conversation lock and bumps the leader
  generation atomically before restoring committed ACK state and hydrating the
  interactive Ink chat history.
- **Background leader**: performs a cooperative stop (`release: false`, never
  `--force`) and requires confirmed process exit before resuming and re-reading
  the final flushed transcript, so concurrent attaches yield a single writer
  and pending inbox messages are neither lost nor duplicated.
- **Live interactive or remote owner / unrecoverable state**: fails closed with
  actionable guidance (including PID when known) without signalling the live
  terminal, without forking, and without mutating durable member state.

### Legacy ACP session without a canonical binding

A legacy ACP session with no canonical `resumeMetadata.swarmBindingV1` is not
recovered by scanning teams or guessing a member. `session/load` accepts one
explicit trusted locator, `_meta.codely.swarmRecovery={teamId,memberId}`, and
only when that pair comes from a deployment receipt or an operator choice. A
missing, malformed, or non-UUID hint fails closed (`recovery_required`) and
does not select a team. When a canonical binding is present, a hint that names
a different team or member is rejected.

Before the cold load, close and verify the exact old ACP host process tree
with the same pid and birth evidence a worker stop uses. Do not edit or delete
conversation lock files to force the load. The legacy conversation owner
treats a held lock as stale only after 30 seconds (`stale: 30_000`, refreshed
every 10 seconds). If the claim is still held, retry the same session id and
the same `swarmRecovery` hint after that window, once the old process is gone.
`session/load` refuses a live owner; it does not fork a new transcript.
`session/new` is the only path that may fork, and it is not leader recovery.

Session load and leader recovery are different acceptance checks:

- Session load claims the conversation lease and reads that transcript. By
  itself it does not bump a generation or settle a stop.
- Leader recovery runs only after that lease is held, for the exact team and
  member named by the canonical binding or the trusted hint. It keeps the same
  tenure and the same conversation session, stops the previous leader state
  when it is not already stopped, and bumps the generation exactly once.
- A heartbeat or stop for the previous fence is rejected. The old operation
  cannot target the new generation.
- The mailbox stays on the member id. Pending messages remain for that
  recovered session and are acknowledged once; recovery does not copy them
  into a second session or replay an already acked message.

### Remote credentials and bounded monitoring

Remote joins use the same bootstrap token and operation ID on retry. The gateway
can rebuild a missing member credential only when its bootstrap hash matches the
association written with the member's initial generation 1 record and the join
configuration still matches. A legacy member without that association is refused;
use an explicit operator-controlled resume/rebind instead of guessing ownership
from a member name. Do not copy tokens or edit the association to bypass refusal.

Control-plane never reissues a member token and does not locally respawn a member
identified as a remote join. Its monitor omits the bootstrap hash. Recovery of
remote credentials and assignments stays with the gateway/remote owner; a local
launch binding is not permission to take over a remote member.

Use loopback HTTP for local gateways. Remote clients require HTTPS, with TLS
termination in front of the loopback gateway; bearer redirects are rejected.
Control-plane's Swarm API is loopback-only by default and needs the explicit
`--allow-swarm-api` opt-in to serve a bind outside that allowlist; that is
independent of gateway tokens. The opt-in API acts with leader authority and
has no operator authentication, so expose it only on a trusted network or
behind an authenticating proxy. Monitor lists and mailboxes show the current page;
navigate explicitly to older/next pages rather than interpreting one page as
the entire team/run. Viewer reconnect replays a bounded live tail with an
omission notice, not the complete owner transcript. Use the canonical transcript
for evidence.

A recorded `pauseOn` result releases its worker but still needs a successor run
for the business decision. Do not resume the admission pool to replay that result.
If pause/cancel reports an identity conflict, re-read the current turn and retry
the control explicitly; do not report it as successfully applied.

### Dispatch, then wait without model polling

After readiness, admission and any required Control-plane visibility check,
give one concise status and end the current model turn. Do not start shell
sleep/status loops or repeatedly inspect worker
sessions while nothing requires a decision. An attached leader process observes
durable task state without requiring a running Control-plane. A Chart run's
terminal state, failed work, human-required pause, or stale owner with active
work produces a durable notification; intermediate Chart node completion does
not wake the leader merely to monitor. No-Chart tasks notify on completion;
idle members without tasks do not imply completion. Pure mailbox collaboration
still uses explicit result messages.

An explicit user cancel keeps automatic model consumption paused. Pending
messages remain recoverable until the user submits a new prompt or explicitly
continues; heartbeat and worker execution are separate. Never acknowledge an
uncommitted message to make cancellation appear successful. Process exit is a
different boundary: without a live leader or Control-plane no observer runs;
reopening the leader can discover durable changes. A durable notification is
only a record in the team store: a detached third-party agent is never woken by
it. Automatic delivery requires an attached leader runtime that drains its own
mailbox; otherwise the agent must query the durable state when it reattaches. A workflow completed event
is not a business pass verdict: inspect the committed result before reporting.

When the checklist above requires coverage, one `codely serve control-plane
--no-rollout` process serves the shared Codely home for every team that uses
that root, independent of repository or current directory. `--swarm-root` is an
explicit isolation
override, not a per-team requirement. Monitoring includes no-Chart teams and
actively joined members; do not equate the workflow run list with the full roster.
The stale-member inspection pass
detects a verifiable incident (stale membership with active work, failed work,
human-required pause), persists one stable attention fact and notifies the
leader/coordinator. This inspection tick is not a workflow tick: stale-member
inspection never starts or resumes a worker. Separately, the enabled idle-wake policy may call the
worker launcher for an eligible parked local member. For unattended leader
re-entry, verify the independently hosted recovery path in the
[deployment recipe](agent-system-deployment.md#unattended-leader-and-member-recovery).
The leader/coordinator inspects the durable task/member/run
state and decides whether, when and how to advance or recover through the
existing public controls (`swarm_manage`, `worker resume/stop`, `workflow
resume/retry`). Whatever action it chooses must preserve generation fences, the
persisted launch/session, session reuse, no replay of completed nodes, and no
recovery after manual stop.
This does not authorize retrying failed business decisions, human-required,
cancelled, stopped or left work, or configuration/approval/schema failures.
An interactive/remote join without a supported launch path remains observable
without being silently converted to a local headless worker.

Keep evidence layers separate. A real main model activated this skill and
generated a frozen review/repair Chart; the Windows Bun executable then ran its
repair and initial-pass branches with a local synthetic provider and real
Control-plane HTTP monitoring. The Windows multi-process gate also passed two
real process cases. Neither result proves real review quality or the separate
cross-machine ACP path. Remote ACP now also has a separate final Linux Bun
attached-host Gate with explicit structured MCP submission, canonical proof,
peer mailbox, viewer detach, gateway restart and old-token fencing. Its final
result contract requires the original full assignment identity; never refresh
it to a newer turn after work. Real browser minimum graph/polling/team-switch/
queued-DM checks passed separately. Full browser fault matrices and every
external generation-recovery combination remain distinct coverage.

Reuse the original run, launch and native session where supported. Before
retrying edits or commands, read durable task/recorder facts. Committed results
must not be replayed; uncertain external effects need an explicit unknown/paused
report rather than a guessed fresh session. Every new worker generation reruns
the original preScripts under the existing launcher.

Report separately: definition saved, members prepared, workers ready, run
admitted, tasks completed, verification passed. Preserve task/member/turn/session
identities, exit codes and evidence paths. End only this run's workers through
the controller; on timeout report cleanup unconfirmed. Never signal an unrelated
or unverified PID. Missing runtime capabilities are “未接线”, not “已支持”.
