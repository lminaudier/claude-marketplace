# lminaudier Plugin

Personal Claude Code extensions. Once installed, skills are invocable as `/lminaudier:<skill-name>`.

## Structure

```
plugins/lminaudier/
├── .claude-plugin/
│   └── plugin.json
├── skills/
│   └── <skill-name>/     ← one directory per skill
│       └── SKILL.md      ← required; frontmatter declares name + description
├── agents/               ← subagent definitions
├── hooks/                ← hook scripts
└── commands/             ← slash command markdown files
```

## Naming conventions

- **Skills**: `skills/commit/SKILL.md` → `/lminaudier:commit`
- **Agents**: `agents/reviewer.md` → available as a subagent type
- **Hooks**: `hooks/pre-tool-use.sh` → runs on the matching Claude Code hook event
- **Commands**: `commands/review.md` → `/lminaudier:review`

## Adding a skill

Create a directory under `skills/` containing a `SKILL.md` with YAML frontmatter:

```markdown
---
name: commit
description: This skill should be used when the user asks to commit changes, create a git commit, or run /commit.
version: 1.0.0
---

# Commit

<instructions for Claude>
```

After adding a skill, run `/plugin update lminaudier@lminaudier-claude-marketplace` to
pick up the new file from the local path.
