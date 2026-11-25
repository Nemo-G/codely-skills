# Local headless and remote on-demand members

Local headless runs one ready work unit per explicit activation. A remote
on-demand host polls the Gateway and starts one agent per accepted Chart
assignment.
Check the installed commands and capabilities before admitting work.

```text
codely swarm headless --help
codely swarm headless prepare --help
codely swarm headless activate --help
codely swarm headless status --help
codely swarm capabilities --team <team> --json
codely serve web-ui --help
```

If the installed version lacks these commands, report that capability gap.
Do not invent flags, edit member JSON, or emulate normal completion with stop.
`execution.headless.localCodely` reports local support on the current host.
`supportedPlatforms` lists `darwin`, `linux` and `win32` for this build.
`activation: "explicit"` and `automaticActivation: false` describe local
activation. `remote: true` advertises the attached on-demand host. The
capability record reports `nativeContext: false` and `externalCli: false`.

## Prepare and explicitly activate a local member

1. Reuse an authorized existing team and choose a new member name. The existing
   `swarm create` command attaches its controller session; it is not a service
   provisioning command to launch and kill.
2. Prepare a `SwarmWorkerLaunchSpec` JSON with `entry: "codely"`, an absolute
   workspace `cwd`, `env`, and `args`. The local installed Codely entry runs the
   child. Preserve requested model/provider settings; omit model overrides when
   none were requested. Discover supported arguments rather than copying flags
   from another executable. Keep secrets in supported SecretRefs.
   Registration resolves `cwd` to its physical path once, then freezes that
   normalized launch. Status reports the pinned path; retargeting the original
   directory alias does not redirect later activations.
3. Run `codely swarm headless prepare --team <team> --member <name>
--launch <JSON> --json`. This saves a headless declaration and frozen launch
   without starting a worker. The member is `dormant`, not process-ready.
4. Deliver authorized work with `codely swarm send --team <team> --member <name>
--kind work --message <text>`, or admit a validated Chart whose member binding
   matches this registered member. Registration is not proof that work ran.
5. Run `codely swarm headless activate --team <team> --member <name> --json`.
   The caller waits for this instance's terminal outcome. It consumes one ready
   work message or workflow turn, including necessary owned BG/tool follow-up.
   Other inbox messages remain pending for later activation.
6. Read `codely swarm headless status --team <team> --member <name> --json`.
   This is a lifecycle/session projection, not a per-message ACK or business
   result reader. Retain the send receipt and activation output. `completed`
   with a successful process outcome means the execution's completion barriers
   passed; inspect its actual work report before claiming the business goal
   passed. For Chart results use `swarm workflow status`; for conversation
   evidence use the installed history tools with the reported session/path.
   `no_work` means no ready unit was executed. Neither outcome means the whole
   DAG is necessarily complete.
7. For the next unit, explicitly activate the same member again. Member/tenure
   and canonical session are retained; generation/instance/fence change. Do not
   prepare a replacement, call the persistent `worker resume`, or fresh-fallback
   when history cannot be restored.

## Verify local completion and recover from interruption

Normal completion commits the selected work, joins its required BG/finalization,
checks recording and ACK/result durability, releases resources and exits.
No normal-success stop/leave or cleanup finalizer is required. Keep identity,
history and results; do not delete the member after each invocation.

An activation response, PID disappearance, model `end_turn`, or a textual
success claim alone is not a business result. `end_turn` while task-owned
background work is still active keeps the claimed workflow owner and waits for
the existing completion notification; it does not fail the task and it does not
rerun the child. With no such background work, missing `swarm_task complete`
proof still fails closed. Read the execution outcome and
the corresponding durable work fact. Failed/cancelled/uncertain activation,
operator-stop, corrupt history, or an unconfirmed old process requires
diagnosis/reconciliation; this slice does not blindly retry it. Report the
reason without changing member JSON or converting the member to persistent.

On interruption, the activation host requests cancellation and verifies exit
of the root and managed descendants against their process birth identities.
If exit is unresolved, a new activation is refused. Continue the same durable
stop operation on the executing host with
`codely swarm worker stop --team <team> --operation <id> --force`.
If no operation exists, target the member with
`codely swarm worker stop --team <team> --member <name> --force`.
A verified local stop receipt for that generation with complete process-tree
coverage allows a subsequent explicit activation of the same eligible member
when the tenure is retained and its canonical session is resumable.

An attended controller can explicitly activate A, observe its terminal outcome,
then explicitly activate B without Control-plane. Each dormant local member
requires a caller to activate it. Control-plane supplies monitoring and
notifications for these members.

## Serve an attached remote member on demand

Use this path for Gateway-ready Chart nodes bound to the joined member.
Mailbox work can be accepted without an assignment and does not start an ACP
agent. Ordinary shared tasks in the same team can enter the assignment
candidate set when they are unbound. The Gateway can record an ordinary claim
and then refuse its execution token with `REMOTE_SWARM_ASSIGNMENT_STALE`,
which stops the host from taking work. Keep ordinary shared tasks in a
separate team or use persistent workers for a mixed backlog.

Use an already joined member and its Gateway-issued token and membership
identity JSON. The host reads both from environment variables. Pass the
variable names and the HTTPS Gateway URL. HTTP is available for loopback.

```text
codely serve web-ui --swarm-on-demand --swarm-url <gateway-url> --swarm-token-env <token-variable> --swarm-identity-env <identity-variable>
```

The host sends membership heartbeats and polls for assignments while idle,
with zero ACP agent processes. Each accepted assignment starts an agent and a
new ACP session under the existing member identity. The host checks durable
assignment completion, closes the agent and confirms the managed process tree
has exited before polling for the next assignment. If tree exit is unresolved,
the host stops taking work. Check the assignment's durable result and the
host's `settled` event with confirmed tree exit before reporting both outcomes.
