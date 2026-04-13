# swiftdiaries-plugins

Personal Claude Code plugin marketplace for development workflows.

## Prerequisites

- [Claude Code](https://claude.ai/code) installed and authenticated
- [beads](https://github.com/swiftdiaries/beads) (`bd` CLI) installed and configured
- A project with `.beads/` initialized (`bd init`)

## Installation

In Claude Code, run:

```
/plugin marketplace add swiftdiaries/swiftdiaries-plugins
/plugin install swiftdiaries@swiftdiaries-plugins
```

Restart Claude Code after install. Skills then surface as `/swiftdiaries:<skill-name>`, e.g. `/swiftdiaries:execute-dev-loop`.

To upgrade:

```
/plugin update swiftdiaries@swiftdiaries-plugins
```

### Local development

To test an unpushed checkout, register the working copy as a marketplace:

```
/plugin marketplace add /absolute/path/to/swiftdiaries-plugins
/plugin install swiftdiaries@swiftdiaries-plugins
```

## Plugins

### swiftdiaries

Skills for structured development workflows.

| Skill | Description |
|-------|-------------|
| `swiftdiaries:execute-dev-loop` | Drive implementation plans to completion through beads task tracking, TDD, parallel subagents, and code review |

## Repo layout

```
swiftdiaries-plugins/
├── .claude-plugin/
│   ├── marketplace.json       # marketplace manifest
│   └── plugin.json            # plugin manifest (same dir — repo root is plugin root)
├── skills/
│   └── <skill-name>/
│       └── SKILL.md
├── LICENSE
└── README.md
```

Single-plugin marketplace: the repo root is simultaneously the marketplace root and the plugin root (`plugins[0].source: "./"`). Top-level `skills/` is discoverable by any tool walking the repo for `SKILL.md` files and by Claude Code's plugin loader.

To add a skill, create `skills/<skill-name>/SKILL.md` conforming to the [Agent Skills spec](https://agentskills.io/specification), then bump `plugin.json` and `marketplace.json` versions.

## License

Apache-2.0. See [LICENSE](LICENSE).
