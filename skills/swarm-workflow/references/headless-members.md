# Explicit headless members

Use this reference for a new local member whose process should run one ready
work unit and exit normally. Existing persistent members keep their current
behavior and commands. First discover the installed surface:

```text
codely swarm headless --help
codely swarm headless prepare --help
codely swarm headless activate --help
codely swarm headless status --help
codely swarm capabilities --team <team> --json
```

If the installed version lacks these commands, report that capability gap.
Do not invent flags, edit member JSON, or emulate normal completion with stop.
The capability result's `execution.headless` describes version and support:
`activation: "explicit"` and `automaticActivation: false` are intentional.
Local Codely support does not imply native-context, remote or external-CLI
support. `supportedPlatforms` is currently `darwin` and `linux`; Windows
prepare/activate rejects before member registration, generation reservation or
child spawn. Status remains readable for diagnosis. Check the installed values
before admitting work.

## Prepare and explicitly activate

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

The first slice supports local Codely execution. Unsupported launch or recovery
combinations fail explicitly; an external CLI name or a remote URL does not
make them supported. Native-context recovery requires a different host and is
not supplied by this entry. Follow the installed validation result.

## Completion and failure

Normal completion commits the selected work, joins its required BG/finalization,
checks recording and ACK/result durability, releases resources and exits.
No normal-success stop/leave or cleanup finalizer is required. Keep identity,
history and results; do not delete the member after each invocation.

An activation response, PID disappearance, model `end_turn`, or a textual
success claim alone is not a business result. Read the execution outcome and
the corresponding durable work fact. Failed/cancelled/uncertain activation,
operator-stop, corrupt history, or an unconfirmed old process requires
diagnosis/reconciliation; this slice does not blindly retry it. Report the
reason without changing member JSON or converting the member to persistent.

## Who drives the next activation?

An attended controller can explicitly activate A, observe its terminal outcome,
then explicitly activate B without Control-plane. Ending a model turn does not
create an activation driver. For unattended DAG advancement, a persistent
driver must implement the headless activation contract; automatic Control-plane
activation is a later #136 capability, not present merely because monitoring
ticks advance. Do not use browser polling or an LLM sleep/status loop as a
replacement scheduler.
