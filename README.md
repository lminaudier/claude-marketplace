# lminaudier Claude Marketplace

Personal Claude Code extension hub by lminaudier.

## Setup on a new machine

```
/plugin marketplace add lminaudier/claude-marketplace
/plugin install lminaudier@lminaudier-claude-marketplace
```

Skills are then invocable as `/lminaudier:<skill-name>`.

> If the repo is private, register via SSH:
> ```
> /plugin marketplace add git@github.com:lminaudier/claude-marketplace.git
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
        ├── skills/               ← skill dirs, each with SKILL.md → /lminaudier:<name>
        ├── agents/               ← subagent definitions
        ├── hooks/                ← hook implementations
        └── commands/             ← slash command markdown files
```

## Adding content

- **Skills**: Create `plugins/lminaudier/skills/<name>/SKILL.md` (directory + file, not a flat .md)
- **Agents**: Add agent definitions to `plugins/lminaudier/agents/`
- **Hooks**: Add hook scripts to `plugins/lminaudier/hooks/`
- **Commands**: Add slash command markdown files to `plugins/lminaudier/commands/`

After adding content, run `/plugin update lminaudier@lminaudier-claude-marketplace` to pick up changes.
