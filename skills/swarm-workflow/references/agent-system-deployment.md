# Configure an agent system on Swarm and Control-plane

Use this reference when an agent, service or automation must configure a durable
team through the installed CLI. All examples use public contracts. No Codely
source checkout, particular chat application or remote host is required.

Apply [the Control-plane decision](../SKILL.md#decide-whether-control-plane-is-required)
before admission: requested monitoring/maintenance, work outliving its initiating
controller, coordination across teams/runs, or unattended repeated/restartable
operation requires coverage. Only a bounded single-team assignment with its
coordinator attached throughout and none of those requirements may use the CLI
alone. Adding a monitor after the controller disappears does not provide
continuous coverage; a DAG label by itself is not the deciding condition.

An attended caller may explicitly activate a new headless member for one work
unit, wait for its terminal outcome, then activate the same member again,
without a Control-plane process. See [headless-members.md](headless-members.md).
This is not unattended scheduling: the current monitor does not automatically
activate headless DAG successors. Preserve the existing persistent worker
deployment when the user has not explicitly requested the new execution mode.

## Assign ownership before starting processes

| Component                      | Owns                                                                                                           | Does not establish                                                                                                      |
| ------------------------------ | -------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------- |
| Coordinating agent/application | User scope, role assignments, frozen definition, task-specific acceptance and integration                      | A second ready queue or fabricated task completion                                                                      |
| Codely Swarm                   | Membership/fences, shared tasks, DAG readiness, claims, messages, committed results                            | That a test assertion is correct or a Git branch is integrated                                                          |
| Execution member               | Its assigned work, original task/run/turn identity and result                                                  | Permission to change peer workspaces or spawn an unrequested team                                                       |
| Deterministic command adapter  | Its declared process, validation and durable business receipt                                                  | Permission to return success after merely launching background work                                                     |
| Control-plane                  | Overview, bounded snapshots/messages, durable incident facts/notifications and the explicit management surface | Choosing or running a recovery strategy, model planning, business retries, feature merges or a chart admission HTTP API |

A reviewer is a responsibility, not a new membership permission. YOLO worker
arguments allow unattended execution under the existing contract; they do not
grant authorization to merge, publish, change credentials or deploy.

## Freeze a deployment record

Keep these values beside the assignment and chart, outside the members' editable
product code. This record is application input, not a new Codely CLI schema.

| Value                                           | How to establish it                                                                                       |
| ----------------------------------------------- | --------------------------------------------------------------------------------------------------------- |
| CLI executable and version                      | Resolve the installed command; run `--version` and `--help` through the same launcher used for workers    |
| Node and bundle, if using the Node distribution | Absolute Node executable plus absolute CLI entry; verify that pair, not another `node`/`codely` on PATH   |
| Shared Swarm root                               | Record the actual CLI home/root and give the same root to workers and the monitor                         |
| Workspace per member                            | Existing absolute directory; record candidate/base SHA when code is involved                              |
| Launch array                                    | One source for `entry`, `cwd`, `env`, `args` and optional `preScripts`                                    |
| Member executor                                 | A sibling of `launch`, never a key inside it: a string, or `{kind:"external-cli",driver}`                 |
| Task artifacts                                  | Original request, definition file/hash, run UUID, team ID/name, prepared members JSON, evidence directory |
| Control-plane                                   | Verified API base URL, matching team ID and configured access boundary; port alone is not identity        |

Prefer the existing default home/root. Explicit isolation must be consistent:
CLI/worker `CODELY_SWARM_ROOT` selects the store; pass that exact absolute path
as the monitor's `--swarm-root`. A different directory is a different task store,
even on the same host. Check a known team ID appears through both interfaces.

`workspaceRef` and `skillRef` are labels, not automatic worktree creation or
skill loading. Create authorized worktrees and prepare dependencies before a
member starts there. Put role instructions in the assignment/prompt or a
verified supported launch argument/environment variable. A role file existing
on disk does not prove it was loaded.

For an npm Node installation, a portable launch has this shape (replace values):

```json
{
  "entry": "/absolute/path/to/node",
  "cwd": "/absolute/existing/worktree",
  "env": {},
  "args": [
    "/absolute/path/to/codely/bundle/gemini.js",
    "--approval-mode",
    "yolo"
  ]
}
```

Use actual Windows paths on Windows, or the installed standalone executable
with no bundle argument. Do not launch a `.cmd`/`.bat` shim as a native executable;
use its supported interpreter or Codely's installed worker launcher. Record PATH
when child tools need Node/npm/git; non-login SSH and service environments may
differ from an interactive shell. Secret values stay outside saved JSON; use
supported secret references and the existing credential owner.

If a host program executes a one-shot CLI operation, use executable + argv,
`shell:false`, bounded captured output, an explicit timeout and closed/ignored
stdin when no input is expected. An unclosed stdin pipe can keep CLI startup
waiting for input. Never construct Swarm's internal fence argv yourself.

## Prepare, verify, admit

This recipe and its portable executable examples describe persistent workers.
For new explicit headless members, use the dedicated headless reference instead:
its launch entry is exactly `codely`, and it resolves the current installation
internally. Registration does not start a process or require readiness.

Read [team-execution.md](team-execution.md) for the create/join lifecycle;
`swarm create` attaches a session, not a fire-and-forget provisioning RPC.
Reuse an existing authorized controller/team when possible. Do not timeout a
create command and assume its membership was rolled back.

1. Discover `worker prepare --help`, `workflow schema --json` and the selected
   CLI's launch flags. Select models only from that environment's discovery.
2. Derive `prepare --members` from the same final chart members used for
   admission: `name = id`, preserve `executor` and the entire `launch`. Do not
   rewrite the file's cwd/entry while preparing from an older in-memory copy.
3. Run `worker prepare`, then `worker status`. Validate readiness and current
   membership, not just a PID. Use bounded startup checks for attach latency.
4. For required monitoring, complete the Control-plane preflight below and
   verify the prepared team's ID before admission.
5. `validate --file ... --json` must return `ok:true`; `plan --team ... --file ...
--json` must return `ok:true` and `data.runnable:true`. Then admit with the saved
   UUID and verify that exact run in Control-plane. Report a real claim/result
   separately from successful admission.

Execution snapshots and physical member launches are frozen at admission.
For authorized live source edits, follow [live-chart-editing.md](live-chart-editing.md):
the public Apply operation updates the actual Chart file and unclaimed work while
preserving owned attempts. A changed launch still needs the existing worker
stop/release/prepare lifecycle; editing JSON alone never reconfigures a process.
If a future checkpoint
needs a workspace that does not exist yet, keep its outer command carrier in a
stable controller directory; the task adapter can prepare the candidate and
then launch an independent inner team. Do not mutate a live member's launch to
point at a new branch.

Example preparation from a materialized chart, using Node without shell quoting:

```js
const members = chart.members.map(({ id, actor, executor, launch }) => ({
  name: id,
  description: actor,
  executor,
  launch,
}));
const result = spawnSync(
  nodeExecutable,
  [
    codelyEntry,
    'swarm',
    'worker',
    'prepare',
    '--team',
    team,
    '--members',
    JSON.stringify(members),
  ],
  { cwd: controllerWorkspace, stdio: 'inherit', shell: false, env },
);
if (result.status !== 0) throw new Error('Worker prepare failed');
```

This is a host-side recipe using `node:child_process`, not a model tool or new
CLI endpoint. Obtain `chart`, paths, team and env from the saved deployment record.

## Required Control-plane preflight

When the decision above requires coverage, reuse a healthy monitor for the same
root. If none exists, start one within the authorized deployment scope; do not
start business work first and promise to add monitoring later:

```text
codely serve control-plane --host 127.0.0.1 --port 4040 --swarm-root <absolute-root> --no-rollout
```

Keep it in the approved process manager or separate terminal. Check
`GET /__codely__/health`, then the known team. `/health` or `/` may return the SPA
HTML and a 200; that is not proof of a healthy Swarm API. Check status,
Content-Type and the JSON response, not the HTTP code alone.

Health is also not proof that the supervisor keeps ticking. The JSON includes
`swarm.supervisor = { running, ticks, lastTickAt }` when a monitor is installed:
read it twice a few seconds apart and require `running === true` and a strictly
increasing `ticks`. A static `ok: true`, a reachable page, or a run visible in
one snapshot proves reachability and data visibility only. If `swarm.supervisor`
is absent, the build predates that signal: treat monitored coverage as
unverified instead of inferring health from the snapshot.

The controller and workers must inherit the same `CODELY_SWARM_ROOT`; the
monitor receives that resolved directory through `--swarm-root`. Keep the monitor
alive independently of the initiating shell/model turn. On a shared host, do not
replace or restart another task's monitor merely to obtain the default port.

### End-to-end observation check

1. Resolve and save the CLI/worker root; inspect an existing monitor's configured
   root or start it with the explicit value above.
2. Query JSON health twice and require `swarm.supervisor.running` plus a
   strictly increasing `ticks`, then query the teams collection. Without that
   progression, HTTP health and a visible snapshot do not prove a live
   supervisor tick. If the target team is not created
   yet, create/prepare it through the normal lifecycle, then verify its actual
   ID through both CLI status and the monitor. Follow pagination when necessary.
3. Only after that preflight, validate/plan/admit the frozen chart using its saved
   run UUID. Query the same team's snapshot with that run UUID and confirm the
   admitted graph appears. Do not require a nonexistent run before admission.
4. Report the monitor URL, store root, team/run IDs and current execution state.
   For an outer checkpoint graph with inner teams, retain both sets of IDs in
   the application's read-only progress snapshot and confirm inner teams appear
   when admitted. A healthy outer team does not prove its child work is visible.
5. End the model turn once dispatch and observation are established. The monitor
   and attached runtime continue without model calls; the monitor only observes
   and notifies, and any recovery decision remains the coordinator's. Do not
   spend model calls polling.

For example, from PowerShell on the host that can reach the monitor (replace
the IDs with the recorded values; these calls only read status):

```powershell
$monitorOrigin = 'http://127.0.0.1:4040'
$monitorApi = "$monitorOrigin/__codely__/control-plane"
$health = Invoke-RestMethod "$monitorOrigin/__codely__/health"
Start-Sleep -Seconds 6
$health2 = Invoke-RestMethod "$monitorOrigin/__codely__/health"
if (-not $health2.swarm.supervisor.running -or
    $health2.swarm.supervisor.ticks -le $health.swarm.supervisor.ticks) {
  throw 'supervisor tick is not advancing; monitored coverage is unverified'
}
Invoke-RestMethod "$monitorApi/swarm/teams?limit=20"
$teamId = '<recorded-team-id>'
$runId = '<admitted-run-uuid>'
Invoke-RestMethod "$monitorApi/swarm/teams/${teamId}?runId=${runId}&limit=20"
```

Other agents may use their HTTP client with the same routes. Treat connection
failure, non-JSON content, missing expected identity, or an incomplete page as a
failed observation check. Do not report monitored readiness from HTTP 200 alone.

### Monitor outage during execution

The underlying workers may keep running while Control-plane is down. Preserve
their identities and effects; do not relaunch, reset their records, or assume the
work was cancelled. Diagnose through the CLI and restore the monitor against the
same root. Report the observation gap and withhold new admissions/resumption
until the team/run snapshot is verified again.

A DAG with running persistent workers can still advance via core task dependencies during an
outage. Dormant headless successors still require a caller to activate them;
admission or monitoring does not supply that caller. If the task requires execution itself to stop when monitoring is lost,
configure an authorized application policy that invokes the existing pause/cancel
control and verifies settlement. Control-plane unavailability does not
automatically pause the graph, and requesting pause can interrupt active work.
Do not claim either behavior without that explicit wiring.

## Monitor API and authority

With the default API base `/__codely__/control-plane`, public Swarm routes are:

| Operation                  | Route / expected use                                                                                                      |
| -------------------------- | ------------------------------------------------------------------------------------------------------------------------- |
| Discover teams             | `GET /swarm/teams?limit=20`; consume `items` and `nextCursor`                                                             |
| Team/run snapshot          | `GET /swarm/teams/<teamId>?runId=<runId>&limit=20`; IDs come from admitted state, not display-name guesses                |
| Member messages            | `GET /swarm/teams/<teamId>/members/<memberId>/messages?limit=20`; follow pagination                                       |
| Send an authorized message | `POST` to that messages route with `{body, kind:"work"}` or `kind:"result"`; 202 is accepted delivery, not completed work |
| Instances                  | `GET /swarm/instances?teamId=<teamId>&limit=20`                                                                           |

These are relative to the API base, not top-level paths. No stock HTTP route in
this contract creates a feature plan, prepares a chart or merges Git branches.
Use the CLI for admission/control. On 404/unsupported, inspect installed help
and report the missing capability; source inspection is not a prerequisite.

Use loopback, or an operator-approved non-loopback bind behind an authenticating
proxy. A bind outside the `127.0.0.1`/`::1`/`localhost` allowlist answers
`/swarm/...` with 403 until the operator passes `--allow-swarm-api`; confirm the
installed help before relying on it. That opt-in acts with leader authority and
adds no operator authentication, so do not enable it just to make an agent's
network request succeed.

After dispatch and the visibility checks, end the coordinator model turn and
rely on durable notifications.
Use a snapshot for an explicit status request or diagnosis. A mechanical
application watcher may observe state without LLM calls; repeated model/shell
sleep loops are not an alternative scheduler.
