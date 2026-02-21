# lminaudier Claude Marketplace

Personal Claude Code extension hub by lminaudier.

## Setup on a new machine

```
/plugin marketplace add lminaudier/claude-marketplace
/plugin install lminaudier@lminaudier-claude-marketplace
```

Skills are then invocable as `/lminaudier:<skill-name>`.

> Before the repo is public, register via local path:
> ```
> /plugin marketplace add ~/Claude
> /plugin install lminaudier@lminaudier-claude-marketplace
> ```

## Structure

```
~/Claude/
├── .claude-plugin/
│   └── marketplace.json          ← marketplace catalog
├── .github/
│   └── workflows/
│       └── validate-plugins.yml  ← CI validation
├── .gitignore
├── README.md
└── plugins/
    └── lminaudier/               ← personal plugin
        ├── .claude-plugin/
        │   └── plugin.json
        ├── README.md
        ├── skills/               ← SKILL.md files → /lminaudier:<name>
        ├── agents/               ← subagent definitions
        ├── hooks/                ← hook implementations
        └── commands/             ← slash command markdown files
```

## Adding content

- **Skills**: Add `*.md` files to `plugins/lminaudier/skills/`
- **Agents**: Add agent definitions to `plugins/lminaudier/agents/`
- **Hooks**: Add hook scripts to `plugins/lminaudier/hooks/`
- **Commands**: Add slash command markdown files to `plugins/lminaudier/commands/`
