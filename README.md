# lminaudier Claude Marketplace

Personal Claude Code extension hub by lminaudier.

## Setup on a new machine

```
/plugin marketplace add git@github.com:lminaudier/claude-marketplace.git
/plugin install lminaudier@lminaudier-claude-marketplace
```

## Commands

| Command | Description |
|---|---|
| `/lminaudier:commit` | Guide through committing staged or unstaged changes using conventional commits, small focused commits, and meaningful "why" explanations. |
| `/lminaudier:copywrite` | Write, draft, review, or improve a business document using Mark Morris's Better Business Writing Skills methodology. |
| `/lminaudier:golden-master` | Augment test coverage on a legacy codebase using the golden master (approval testing) technique — capture current behaviour as snapshots, then use code coverage to find untested paths and expand inputs until the safety net is solid. |

## Skills

Skills are model-invocable — Claude applies them automatically when relevant, no slash command needed.

| Skill | Description |
|---|---|
| `ast-grep` | Structural code search using AST patterns. Activated when searching for code constructs beyond what text search can handle. |
| `bear` | Interact with Bear notes via `bcli`. Activated when the user mentions Bear notes, wants to find, read, create, edit, or manage notes and todos. |

## Dependencies

Some skills require external tools to be installed on the machine.

| Tool | Required by | Install |
|---|---|---|
| [`ast-grep`](https://ast-grep.github.io) | `ast-grep` skill | `brew install ast-grep` |
| [`bcli`](https://github.com/mreider/better-bear-cli) | `bear` skill | `brew install mreider/tap/bcli` |
