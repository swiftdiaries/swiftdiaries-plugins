# swiftdiaries-plugins

Personal Claude Code plugin marketplace for development workflows.

## Prerequisites

- [Claude Code](https://claude.ai/code) installed and authenticated
- [beads](https://github.com/swiftdiaries/beads) (`bd` CLI) installed and configured
- A project with `.beads/` initialized (`bd init`)

## Installation

Add the marketplace and install the plugin:

```
/plugin marketplace add github:swiftdiaries/swiftdiaries-plugins
/plugin install swiftdiaries
```

Or test locally during development:

```bash
claude --plugin-dir ./swiftdiaries
```

## Plugins

### swiftdiaries

Skills for structured development workflows.

| Skill | Description |
|-------|-------------|
| `swiftdiaries:execute-dev-loop` | Drive implementation plans to completion through beads task tracking, TDD, parallel subagents, and code review |

## Adding Skills

```
swiftdiaries-plugins/
├── .claude-plugin/
│   └── marketplace.json
├── swiftdiaries/
│   ├── .claude-plugin/
│   │   └── plugin.json
│   └── skills/
│       └── <skill-name>/
│           └── SKILL.md
└── README.md
```

Add new skills under `swiftdiaries/skills/<skill-name>/SKILL.md`, then update the marketplace version.
