# lminaudier Claude Marketplace

Personal Claude Code extension hub by lminaudier.

## Setup on a new machine

```
/plugin marketplace add git@github.com:lminaudier/claude-marketplace.git
/plugin install lminaudier@lminaudier-claude-marketplace
```

## Available extensions

- **Skills** (`plugins/lminaudier/skills/`) — context injected into Claude automatically; invoked implicitly based on description
- **Commands** (`plugins/lminaudier/commands/`) — slash commands invocable as `/lminaudier:<name>`
- **Agents** (`plugins/lminaudier/agents/`) — subagent definitions
- **Hooks** (`plugins/lminaudier/hooks/`) — scripts that run on Claude Code hook events
