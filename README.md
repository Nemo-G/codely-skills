# Codely skills

A multi-skill repository for operating Codely through its public CLI and
Control-plane. Each skill lives in its own sibling folder under `skills/`, so
users can install one skill or the complete Codely skill suite from this single
repository.

## Prerequisites

Install one Codely release lane before using these skills. Do not install the RC
and Nightly lanes side by side.

### RC lane

The public npm package publishes the release-candidate line on its `latest`
dist-tag and requires Node.js 20 or newer:

```sh
npm install -g @unity-china/codely-cli@latest
```

### Nightly lane

Nightly is the daily snapshot lane. It has not completed the RC lane's full QA,
regression, or stability verification. The Nightly documentation recommends
Node.js 26.4 or newer and requires removing the public npm installation before
running the platform installer.

Windows PowerShell:

```powershell
npm remove -g @unity-china/codely-cli
iex (iwr "https://codesearch-plugins.tos-cn-shanghai.volces.com/codely-cli/install.ps1" -UseBasicParsing)
```

Linux or macOS:

```sh
npm remove -g @unity-china/codely-cli
curl -fsSL "https://codesearch-plugins.tos-cn-shanghai.volces.com/codely-cli/install.sh" | bash
```

See the [npm package](https://www.npmjs.com/package/@unity-china/codely-cli)
and the official [Codely CLI Nightly guide](https://codely-docs.tuanjie.cn/en/using-codely/codely-cli-nightly/)
for the current published versions and platform notes.

### Verify installation and authentication

Whichever lane you choose, run the same probe:

```sh
codely --version
codely --cmd model
```

`codely --cmd model` is both a capability and authentication check: it requires
Codely to load the current credential and fetch the available model catalog. If
it succeeds, an existing cached login or `CODELY_TOKEN` is already usable; skip
authentication setup.

If `codely` is not found, repair the selected installation lane and rerun the
probe. If the model probe fails specifically because authentication is missing:

- In a non-interactive or headless environment, provide `CODELY_TOKEN` through
  the process environment, then rerun `codely --cmd model`. Never print, commit,
  or place the token in a Chart, prompt, or command-line argument.
- In an interactive terminal, launch `codely`, enter `/auth`, complete Codely
  authentication, exit the session, then rerun `codely --cmd model`.

Do not replace credentials merely because the probe reports a network or model
catalog error; preserve the original error and resolve that failure first.

## Install skills

```sh
npx skills add Nemo-G/codely-skills --skill swarm-workflow
```

Add `-g` for a user-level installation. To discover the available skills without
installing:

```sh
npx skills add Nemo-G/codely-skills --list
```

Install every skill in the repository with:

```sh
npx skills add Nemo-G/codely-skills --all
```

## Repository layout

```text
codely-skills/
├── skills/
│   ├── swarm-workflow/
│   │   ├── SKILL.md
│   │   ├── examples/
│   │   └── references/
│   └── another-skill/
│       └── SKILL.md
├── LICENSE
├── NOTICE
└── README.md
```

Add future skills as sibling directories such as `skills/<skill-name>/`; do not
place unrelated skill entrypoints inside `skills/swarm-workflow/`. Every skill
owns its `SKILL.md` and any optional `agents/`, `scripts/`, `references/`,
`examples/`, or `assets/` resources it actually needs.

Do not add an `allowed-tools`/`allowedTools` allowlist by default. Third-party
agents expose different tool names and capability models, so a portable skill
must not assume one host's internal tool schema. Let the host provide its normal
tool set and have the skill discover the concrete commands and capabilities at
runtime. Add a tool restriction only when a specific target platform requires a
documented, portable restriction contract.

## swarm-workflow

Organize durable Codely teams, assign analyzer/implementer/reviewer roles, run
ordered workflows, and open the published interactive Workbench. The skill also
covers launch bindings, explicit headless activations, monitoring, and recovery.

The skill is documentation; it does not bundle Codely, credentials, a scheduler,
or the Workbench implementation. Install Codely separately. Before executing a
workflow, verify that your installed version exposes the required commands:

```sh
codely --version
codely swarm --help
codely swarm worker prepare --help
codely swarm workflow schema --json
codely swarm workflow dashboard --help
codely serve control-plane --help
```

Use the installed schema and capability discovery as the compatibility contract.
No universal minimum version is asserted: this public snapshot has not been
qualified against a matrix of released Codely versions. In particular, explicit
headless support is platform-dependent; follow the installed capability response.

Read [the skill](skills/swarm-workflow/SKILL.md) for the workflow and routed
references. The three example Charts contain placeholders that must be replaced
with real, authorized task inputs before execution. `skillRef` labels do not
automatically load role instructions.

## License and provenance

Apache-2.0; see [LICENSE](LICENSE) and [NOTICE](NOTICE). This is a standalone
distribution of the Codely `swarm-workflow` skill and its referenced resources.
