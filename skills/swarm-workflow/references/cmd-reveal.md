# Codely reveal surfaces (do not guess)

Use these **before** inventing model ids, auth, tools, startup fields, or
`WorkflowDefinition` keys. Prefer live `--help` / list / `--cmd` output over
memory. When unsure, start with `codely --help`.

## 1. Instant list/exit argv (no model turn)

```powershell
codely --help
codely --version
codely --list-tools
codely --list-extensions          # alias -l
codely --list-jobs
codely --list-example-prompts
```

## 2. Provider / model probing (pair with `--cmd`)

```powershell
codely --cmd "/model"                           # current auth's /v1/models
codely --cmd "model list"                       # alias of list-provider
codely --cmd "/model list-provider"
```

Default discovery does not add `--auth`; it uses the current login/configuration.
Only probe another provider when the user explicitly requests that auth override.
An authentication failure is not permission to change credentials or fabricate
a token. Do not run `/model use` to mutate shared workspace settings for workers.

Common pairing flags (always confirm via `codely --help`):

| Flag                                       | Role                                                             |
| ------------------------------------------ | ---------------------------------------------------------------- |
| `--auth` / `--auth-type`                   | Provider auth for this process (`CUSTOM_AUTH=1` for non-default) |
| `-m` / `--model`                           | Main model slot                                                  |
| `--flash`                                  | Flash / internal slot                                            |
| `--multimodal`                             | Multimedia analysis slot                                         |
| `--default-agent-model`                    | Default for task-spawned subagents                               |
| `--openai-api-key` / `--openai-base-url`   | Custom OpenAI-compatible endpoint                                |
| `--wire-api`                               | `chat` \| `responses` \| `messages`                              |
| `--reasoning_effort`                       | `minimal`…`max` (see `--help` choices)                           |
| `--enable_thinking` / `--enable_reasoning` | chat_template_kwargs.toggle                                      |
| `--extra_body`                             | JSON object string merged into request                           |
| `--proxy`                                  | HTTP(S) proxy                                                    |

Do **not** invent model ids absent from `/model` output for that auth.

## 3. Headless slash via `--cmd`

`--cmd` runs a slash command without the TUI (`/` optional). Example:

```powershell
codely --cmd "/tools"
codely --cmd "/tools desc"
codely --cmd "/skills"
codely --cmd "/mcp"
codely --cmd "/agents list"
codely --cmd "/commands list"
codely --cmd "/memory show"
codely --cmd "/stats"
codely --cmd "/usage"
codely --cmd "/lsp status"
codely --cmd "/upm status"
```

Swarm status is a standalone command, not a headless slash command:

```powershell
codely swarm status --team <team> --limit 20
codely swarm events --team <team> --limit 50 --json
codely swarm events --team <team> --after <journalId>:<sequence> --expand --json
```

Dialog-only actions (`/model settings`, theme pickers, etc.) are poor fits for
`--cmd`. Prefer list/status style commands.

`--cmd` cannot combine with `-p` / `-i` / `--example-prompt` / resume / sandbox
(see parser errors). YOLO may be required for some mutating slash actions.

## 4. Session / policy flags that change what is visible

Reveal or constrain the tool/skill surface for a run:

`swarm-workflow` names a builtin skill, not a command under `codely` or
`codely swarm`. Start an enabled session with `--experimental-swarm`, inspect its
`/skills` catalog, and use the model's `activate_skill` tool with that name.
The standalone `codely skills list` enumerates filesystem/extension skills and
is not proof that a gated builtin is visible in the current model session.

```text
codely --experimental-swarm --cmd "/skills"
```

| Flag                                                           | Effect on discovery                                              |
| -------------------------------------------------------------- | ---------------------------------------------------------------- |
| `/swarm`                                                       | Activates swarm tools in an interactive non-ACP session          |
| `--experimental-swarm`                                         | Enables Swarm tools and the gated builtin skill for this process |
| `--experimental-memory`                                        | Enables structured memory tools + `/memory`                      |
| `--experimental-lsp`                                           | Enables `lsp` tool + `/lsp`                                      |
| `--experimental-dsh` (+ `--dsh-profile`, `--assembly-backend`) | DSH assembly / tool set                                          |
| `--experimental-skills` / `--no-experimental-skills`           | Skills on/off (default on)                                       |
| `--skills a,b`                                                 | Limit enabled skills                                             |
| `--agents` / `--no-agents`                                     | Agents system on/off                                             |
| `--agent <name>`                                               | Start with selected subagent                                     |
| `--allowed-mcp-server-names`                                   | Restrict MCP servers                                             |
| `--extensions` / `-e`                                          | Restrict extensions                                              |
| `--exclude-tools`                                              | Hide tools for this session                                      |
| `--collaboration-mode`                                         | `default` \| `plan` \| `ask` (capability envelope)               |
| `--approval-mode` / `-y`                                       | Approval / YOLO (needed for some `--cmd` mutations)              |
| `--path-policy`                                                | Workspace path boundary                                          |
| `--output-format`                                              | `text` \| `json` \| `stream-json` for non-interactive replies    |

Feature flags do not replace list commands — after enabling, re-run
`--list-tools` / `--cmd "/tools"` / `--cmd "/skills"`.

## 5. Management subcommands (self-describing)

Each has its own `--help` tree:

```powershell
codely skills --help          # list / enable / disable / install / uninstall
codely mcp --help             # add / remove / list / enable / disable / auth
codely agents --help
codely extensions --help
codely commands --help
codely update --help
codely serve --help           # Unity/MCP serve surface
```

Examples:

```powershell
codely skills list
codely mcp list
codely agents list
codely extensions list
```

## 6. Swarm / WorkflowDefinition (chart ground truth)

Loaded only when argv contains `swarm`:

```powershell
codely swarm --help
codely swarm workflow --help
codely swarm workflow schema --json
codely swarm capabilities --team <team> --json
codely swarm workflow examples --json
codely swarm workflow validate --file <file> --json
codely swarm workflow plan --team <team> --file <file> --json
codely swarm workflow list --team <team> [--cursor <cursor>] [--limit <n>] [--json]
codely swarm workflow run|status|pause|resume|cancel|retry …
codely swarm workflow dashboard --team <team> --file <file> --run-id <id> --output <file> [--port 0] [--no-serve]
codely swarm template save --chart-file <file> [--name <name>] [--description <text>] [--force] [--json]
codely swarm template list [--json]
codely swarm template show <name> [--json]
codely swarm template instantiate <name> [--inputs <file.json>] [--json]
codely swarm template run <name> --team <team> --run-id <uuid> [--inputs <file.json>] [--json]
codely swarm create|join|list|status|send|stop|worker …
```

- `template instantiate` returns the exact definition (and member launch
  bindings) `template run` admits for the same inputs. Prepare members with
  those bindings; admission keeps the bound path, so a launch rebuilt from the
  managed package fails closed as a conflicting execution binding.
- One-shot `codely swarm workflow …` does **not** need `--experimental-swarm`.
- In an interactive non-ACP session, `/swarm` activates the swarm tools after
  confirming the team configuration. It accepts no subcommand arguments.
- `schema --json` is the only allowed-key list for `WorkflowDefinition`
  (including `members[].launch`). On validate unknown-key errors, re-run
  schema — IR may have drifted.
- `capabilities` is roster + static catalog (`probes` false). Live model ids
  still come from `--cmd "/model"` (no auth override by default).
- `workflow list` returns `runId`, `definitionHash` when present, `status` and
  `nextCursor` for one team only (default page 50, maximum 100). It is read-only,
  works out of session, and never needs a Control-plane. Feed a listed `runId`
  to the existing `workflow status`; do not derive a run id from a hash.

## 7. Suggested explore order for this skill

1. `codely --help` — see flags
2. Model ids: `--cmd "/model"`; `--auth …` only if explicitly requested
3. Tools/skills/mcp: `--list-tools`, `--cmd "/tools"`, `skills list`, `mcp list`
4. Workflow IR: `swarm workflow --help` → `schema --json`

5. Author JSON → `validate` → `plan` → `run`
6. Reuse: `swarm list`/`status` → `workflow list` → `workflow status`, then
   `worker status|resume|prepare` for each member. These CLI operations do not
   technically need Control-plane; execution still follows
   [the monitoring decision](../SKILL.md#decide-whether-control-plane-is-required).

For team provisioning and readiness, read [team-execution.md](team-execution.md).
For a host agent configuring processes and a shared monitor, read
[agent-system-deployment.md](agent-system-deployment.md). For executable graph
shapes and recovery choices, read [complex-workflows.md](complex-workflows.md).
Discovery exposes available interfaces; it does not create workers or prove that
they are ready. Explore only surfaces relevant to the request, not every command
in this reference on every run.
