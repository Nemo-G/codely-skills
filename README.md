# Codely skills

A multi-skill repository for operating Codely through its public CLI and
Control-plane. Each skill lives in its own sibling folder under `skills/`, so
users can install one skill or the complete Codely skill suite from this single
repository.

## Install

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
