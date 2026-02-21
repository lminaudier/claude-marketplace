# lminaudier Plugin

Personal Claude Code extensions. Once installed, skills are invocable as `/lminaudier:<skill-name>`.

## Structure

```
plugins/lminaudier/
├── .claude-plugin/
│   └── plugin.json
├── skills/       ← SKILL.md files (each becomes /lminaudier:<filename-without-ext>)
├── agents/       ← subagent definitions
├── hooks/        ← hook scripts
└── commands/     ← slash command markdown files
```

## Naming conventions

- **Skills**: `skills/commit.md` → `/lminaudier:commit`
- **Agents**: `agents/reviewer.md` → available as a subagent type
- **Hooks**: `hooks/pre-tool-use.sh` → runs on the matching Claude Code hook event
- **Commands**: `commands/review.md` → `/lminaudier:review`

## Adding a skill

Create a markdown file in `skills/` following the SKILL.md format:

```markdown
# Skill: <name>

<description of what the skill does>

## Instructions

<detailed instructions for Claude to follow>
```
