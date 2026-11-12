# Independent live Chart HTML

Use for a user-requested clickable DAG, live monitoring, Raw Chart editing or
interactive node configuration. Requires only the installed Codely CLI and its
public Control-plane API. The consumer does not need a source checkout.

## Deliver a page for this team

Discover `codely swarm workflow dashboard --help` and `schema --json`. Preserve
the user's actual file, shared root, team and admitted run. Generate once:

```text
codely swarm workflow dashboard --team <team> --file <absolute-chart.json> --run-id <uuid> --output <task-owned/team.html> --open
```

The default HTML/CSS/JS chart template ships once with the installed CLI
(`swarm-chart.html`, `swarm-chart.js`, `swarm-chart.css` under the published Web
UI assets). The command mechanically copies that published template to
`--output` and writes `<output>.binding.json`; the only per-team artifact is the
binding. It never generates per-team business UI and never copies implementation
code into the page. The template is not a source tree the skill edits.

The binding file is the unit of registration: server-side registration is
explicit binding registration and remains the permission boundary; the browser
cannot choose arbitrary filesystem paths. Keep the process running in the task's
normal process manager. It uses the same Swarm root as the CLI. `--port 0`
selects a free port and prints the actual URL; open that printed URL and do not
hardcode or assume a port. Do not replace another team's listener to obtain a
particular port.

For several independent pages on one server, generate each with `--no-serve`,
then supply their binding files to one process:

```text
codely serve control-plane --host 127.0.0.1 --swarm-root <root> --no-rollout --swarm-chart <a.html.binding.json> --swarm-chart <b.html.binding.json>
```

On restart reuse these files; do not overwrite/regenerate their Chart from prose.
Run the existing supervisor-health and team/run identity checks. The all-team
Control-plane UI is available as a reference, but link the **independent team HTML**
as the deliverable. Open it and exercise a node click and validation before handing
it over. Make edits to a live task only within the user's authorized scope.

Generic template guidance must not hardcode a business run: no MR/issue number,
listener port, machine path, business stage or internal store path belongs in the
template or these instructions. Examples use placeholders such as `<team>`,
`<absolute-chart.json>`, `<root>` and `<uuid>`; the CLI resolves real roots and
durable records from its own configuration, never by hand-assembled page data.

Inspection keeps facts, not policy: it persists the observed incident facts and
notifies the leader/coordinator, which decides whether, when and how to tick or
recover. The page does not scan logs, add a second execution source, or
auto-recover. A `resume` + `retry` sequence is one sample recovery path, not the
only valid strategy or order.

The team snapshot also carries a read-only `coordination` projection: the
supervisor scan ticks, the coordination reminders the tick actually wrote
(created/queued, delivered and ACKed, plus the receiver's own after-turn fact),
the processing phase — `unknown` until that after-turn fact exists, `ended`
once it does — and each pending task's real owner/status. The page renders those fields instead
of deriving a state of its own; "pending work with 0 running" is shown as 待协调,
and a queued or ACKed reminder is shown as waiting for the leader, never as a
reason to send a new wake-up. A failed or unreadable mailbox/task read is shown
as unknown instead of "no reminder", exact ids stay collapsed behind a
disclosure, and owner/run names are displayed through the roster rather than as
raw UUIDs. The projection is not a second scheduler and never writes history.

## Template, binding and browser config

The published template carries exactly one binding placeholder inside the frozen
config script `<script type="application/json" id="codely-swarm-chart-binding">`.
The server replaces it with minimal browser config only (`chartId`, `apiBase`,
optional `initialTab`/`uiCapabilities`) and escapes `<` so the config cannot
close the script element. Team/run/file paths stay in the server-side binding and
never reach the browser. Old `.binding.json` files without the new optional
fields remain valid and are never rewritten.

## Direct editing contract

- The raw source text is the single edit fact source. `GET /swarm/charts/<id>`
  returns the exact raw bytes and their `fileRevision`.
- Structured edits splice token spans in that raw text. Never rebuild the parsed
  object and never use whole-object JSON.stringify; only the explicitly edited
  value span may change, and all unmatched tokens stay byte-for-byte identical.
- Unmatched content survives edited sibling fields: unknown fields, explicit
  `null`, empty arrays, array order, key order, big integers (for example
  `9007199254740993`), exponent/negative-zero forms, Unicode escapes and a data
  key named `__proto__`. The browser never injects or fills defaults while
  viewing, polling, selecting or expanding a node.
- A JSON sub-field editor validates that its text is one JSON value and splices
  that text. Empty text requires an explicit user choice between "delete field"
  and "set null"; those are different results.
- Source raw bytes, the current source version (`fileRevision`,
  `definitionHash`) and the immutable execution attempt snapshot are displayed
  side by side. They are never merged or treated as one version, and an old
  attempt's result is never painted onto a newer source.
- Execution status, test evidence and review verdict are separate. Historical
  evidence and the current candidate are grouped by their explicit source
  version/receipt identity. An external CLI process is an executor configuration,
  not a separate durable member; only the durable roster is a member list.
  When the evidence fields are absent, show "undeclared" instead of guessing
  PASS from node names, actor names, exit codes or file existence.
- The page supports any DAG: fork, join, orphan and cross-layer edges, with
  native SVG/DOM edges and deterministic layout. Node/member forms, dependency
  editing, Raw JSON, Validate, Apply and the conflict path remain real visual
  capabilities; Raw is a full editing capability, not a substitute for the
  structured node, member, execution or evidence views.
- Unknown fields the current server schema rejects still cause Validate to fail
  with its path; that is a Validate failure, not silent deletion, and the draft
  text is preserved. A file changed by another editor returns a conflict; the
  user's draft and selection are retained for reconciliation.

## Validate, Apply and concurrency

- **Validate** checks without writing. **Apply & Validate** validates, checks the
  file revision, writes the original file, then revises the admitted pending work.
  Read its actual receipt; a file save or HTTP acceptance alone is not completion.
- An Apply request freezes `submitted = {raw, baseFileRevision, scope}` and sends
  `{raw, expectedFileRevision: submitted.baseFileRevision}`. The expected revision
  is always the submitted raw plus baseFileRevision, never a newer polled server
  revision.
- A late Apply response must never overwrite a newer current draft or the user's
  selection; it always describes the submitted raw. If the draft is unchanged,
  update the base and mark it clean. If the user kept typing on the same lineage,
  advance only the base revision and keep the draft dirty and its patch history.
  If Reload or a scope change replaced the lineage, the old response changes
  nothing.
- On `409 CHART_FILE_CONFLICT` (or a blocked revision/task code), keep the draft,
  selection and history, show the latest server source in the compare area, and
  never auto-rebase or replay. A timeout is "result unconfirmed": do not resend
  the POST, and do not mark the draft clean.
- Invalid input stays in the editor and is not written. Unclaimed nodes take the
  revision on their next dispatch. An owned/finished attempt retains its identity,
  configuration and result; its edited configuration is recorded for a subsequent
  explicit attempt, not retroactively reported as used.
- Adding/removing an unclaimed node changes future scheduling. Completed results
  remain evidence for their original definition, never for a newly edited one.
- Member launch changes still follow the existing worker lifecycle: inspect
  readiness and use the documented stop/release/prepare order when a launch binding
  changes. Changing a file cannot replace a running process's model or cwd.

## Refresh, disconnection and feedback

Browser polling only reads state and never overwrites a dirty draft or the
selection. A clean poll may take the server's new raw revision; a dirty poll
updates only the server/compare view. Chart and run reads succeed independently:
keep the last successful data, mark each resource stale on its own, and show its
last success time and backoff. Stop polling on a hidden page and refresh
immediately when visible. Automatic refresh runs once per minute (60,000ms) by
default, a failed read never retries faster than that interval, and the next read
is only scheduled after the previous one finished; manual Reload stays
immediate. Disconnection never replays a POST, and the page provides no execution
queue and never starts a tick on its own.

Use the existing member mailbox route for authorized human feedback. Show the
202 receipt's `delivery` and `targetGeneration` values, and say explicitly that
accepted/queued is not processed or completed. Closing the page does not stop
the Swarm.

## Public API

Base: `/__codely__/control-plane`. Bindings are explicitly supplied by the CLI;
the browser cannot choose arbitrary filesystem paths.

| Request                                                  | Result                                                                |
| -------------------------------------------------------- | --------------------------------------------------------------------- |
| `GET /swarm/charts`                                      | Registered charts and supported operations                            |
| `GET /swarm/charts/<id>`                                 | Original raw file, byte revision, schema, complete run node summaries |
| `GET /swarm/charts/<id>/page`                            | The generated team HTML                                               |
| `GET /swarm/charts/<id>/nodes/<nodeId>`                  | Attempt configuration, committed output/reference, events             |
| `POST /swarm/charts/<id>/validate`                       | Validate `{raw}` without side effects                                 |
| `POST /swarm/charts/<id>/apply`                          | Apply `{raw, expectedFileRevision}` and return per-node effects       |
| `GET /swarm/teams/<teamId>?runId=<runId>`                | Durable roster, execution status, events, artifacts                   |
| `POST /swarm/teams/<teamId>/members/<memberId>/messages` | Mailbox feedback with a 202 receipt                                   |

The browser composes these existing endpoints in one snapshot adapter; it does
not add a second snapshot store, log scan or execution scheduler. Additive
response fields are copied from the same durable task record. Reads and writes
use the existing local API boundary. No arbitrary shell endpoint.

CLI equivalent for scripted clients:

```text
codely swarm workflow read --team <name> --file <chart> --run-id <uuid> --json
codely swarm workflow apply --team <name> --file <chart> --run-id <uuid> --from <edited-json> --expected-file-revision <read-revision> --json
```

If the installed command or API is absent, report the required version/capability.
Do not silently simulate Apply by editing internal Swarm records or merely saving
a second draft file.
