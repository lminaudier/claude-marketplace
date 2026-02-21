---
name: lminaudier:commit
description: Guide through committing staged or unstaged changes using conventional commits, small focused commits, and meaningful "why" explanations.
argument-hint: [optional context or message hint]
allowed-tools:
  - Bash
  - Read
  - AskUserQuestion
---

Guide the user through committing code changes to git. Follow these principles strictly.

## Conventional Commits format

Every commit message MUST follow this structure:
```
<type>(<optional scope>): <short description>

<optional body explaining WHY — not what>
```

**Types:**
- `feat` — a new feature
- `fix` — a bug fix
- `refactor` — code restructuring without behavior change
- `chore` — tooling, dependencies, config, build
- `docs` — documentation only
- `test` — adding or updating tests
- `perf` — performance improvement
- `style` — formatting, whitespace (no logic change)
- `ci` — CI/CD pipeline changes
- `revert` — reverting a previous commit

**Rules for the short description:**
- Lowercase, no trailing period
- Imperative mood ("add", "fix", "remove" — not "added", "fixes", "removed")
- Max ~72 characters for the full first line

**The body must explain WHY:**
- Why was this change necessary?
- What problem does it solve?
- What was the trade-off or decision made?
- Never just describe what the diff shows — the diff already shows that.

## Small, focused commits

Before committing, inspect the staged/unstaged changes:
1. Run `git status` and `git diff --stat HEAD` to understand the scope
2. If the changes span multiple unrelated concerns, split them into separate commits
3. Each commit should represent one logical change that can be understood, reviewed, and reverted independently
4. A good rule of thumb: if the commit message needs "and" to describe what changed, it should probably be two commits

## Workflow

1. Run `git status` to see what's staged and unstaged
2. Run `git diff` (and `git diff --cached` if there are staged changes) to read the actual changes
3. Assess whether the changes form a single logical unit or should be split
4. If splitting is warranted, tell the user what groupings you suggest and why — ask for confirmation before staging selectively
5. Draft a commit message following the conventional commits format
6. Explain the "why" you inferred from the diff — ask the user to confirm or correct your interpretation before committing
7. Execute the commit using a HEREDOC to preserve formatting:
   ```bash
   git commit -m "$(cat <<'EOF'
   type(scope): short description

   Body explaining why this change was made.
   EOF
   )"
   ```
8. Show the result of `git log --oneline -3` so the user can confirm the commit looks right

## Guardrails

- Never commit files that likely contain secrets (.env, credentials, private keys). Warn the user if any are staged.
- Never use `--no-verify` unless the user explicitly requests it.
- Never amend a commit that has already been pushed to a remote.
- Prefer `git add <specific-files>` over `git add -A` or `git add .` to avoid accidentally staging unintended files.
- If a pre-commit hook fails, fix the underlying issue rather than bypassing the hook.
