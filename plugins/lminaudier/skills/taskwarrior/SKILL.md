---
name: taskwarrior
description: >-
  Interact with Taskwarrior using the task CLI.
  Use this skill whenever the user mentions Taskwarrior, tasks, todos, a to-do
  list, or wants to add, list, complete, modify, filter, or manage tasks.
  Activates on requests like "add a task for X", "what are my tasks today",
  "mark task done", "show overdue tasks", "show my next tasks", "create a
  recurring task", "list tasks for project Y", "what's blocked", etc.
allowed-tools:
  - Bash
---

# Taskwarrior -- task CLI

Taskwarrior is a local, file-based task manager. All data lives in `~/.task/`.
No network calls, no accounts.

## Guardrails

- **Read before modify**: run `task <id> info` before proposing changes to a
  task. Never guess at task IDs -- always confirm with a list or filter first.
- **No blind deletes**: never `delete` a task without showing it to the user
  first and getting explicit confirmation. Prefer `done` over `delete`.
- **Filters before bulk ops**: always preview what a filter matches before
  applying a `modify` or `done` across multiple tasks.
- **Escape shell specials**: descriptions and annotations with `!`, `(`, `)`,
  `*`, etc. must be quoted or escaped to avoid shell expansion.
- **Use `--` for ambiguous descriptions**: if a description looks like a
  filter expression, use `task add -- <description>` to prevent
  misinterpretation.

## Quick Reference

| Task | Command |
|------|---------|
| Add a task | `task add Buy milk` |
| Add with project & priority | `task add +work project:Home priority:H Fix roof` |
| Add with due date | `task add due:tomorrow Call dentist` |
| List next/most urgent | `task next` |
| List all pending | `task list` |
| List ready tasks | `task ready` |
| Show overdue | `task overdue` |
| Show task details | `task <id> info` |
| Mark done | `task <id> done` |
| Delete a task | `task <id> delete` |
| Modify a task | `task <id> modify due:friday priority:H` |
| Add annotation | `task <id> annotate "Call back at 3pm"` |
| Start a task | `task <id> start` |
| Stop a task | `task <id> stop` |
| List projects | `task projects` |
| List tags | `task tags` |
| Show summary by project | `task summary` |
| Export as JSON | `task export` |

## Command Discovery

Taskwarrior is self-documenting:

```bash
task help           # Full command and attribute reference
task commands       # List all commands with descriptions
task columns        # All supported columns and format styles
```

Always use `task help` to verify flags rather than guessing.

## Adding Tasks

```bash
task add Buy milk
task add +work project:Acme priority:H "Write proposal"
task add due:tomorrow scheduled:today +errand Pick up dry cleaning
task add recur:weekly due:monday "Weekly team sync"
task add wait:2d "Follow up with client"    # hidden until 2 days from now
task add depends:42 "Deploy after tests pass"
task add -- project:Home needs scheduling   # -- forces rest as description
```

**Priority values:** `H` (high), `M` (medium), `L` (low). Omit for none.

**Date shortcuts:** `today`, `tomorrow`, `yesterday`, `eow` (end of week),
`eom` (end of month), `eoy` (end of year), `monday`…`sunday`, `now`,
`+2d`, `+1w`, `2026-04-30`.

## Listing and Filtering

```bash
task list                          # All pending tasks
task next                          # Most urgent (by urgency score)
task ready                         # Actionable (not blocked, not waiting)
task overdue                       # Past due date
task all                           # All tasks including completed/deleted

# Filter by attribute
task project:Home list
task +work list
task priority:H list
task due:today list
task due.before:eow list

# Filter by status
task status:waiting list
task status:completed list

# Combine filters (AND by default)
task project:Acme +work priority.not:L list

# Algebraic / OR
task '(project:Home or project:Garden)' list

# Count matching tasks
task project:Acme count
```

## Modifying Tasks

```bash
task <id> modify priority:H
task <id> modify due:friday project:Work
task <id> modify +urgent -someday
task <id> modify description:"Updated task name"

# Bulk modify (always preview filter first)
task project:Home list                        # preview
task project:Home modify +review              # then apply
```

## Completing and Deleting

```bash
task <id> done              # mark complete
task <id> done +reviewed    # mark complete and add tag at once
task <id> delete            # soft-delete (ask user first)

# Bulk complete
task +errand done           # use filter instead of ID for bulk
```

## Annotations

```bash
task <id> annotate "Waiting for reply from Alice"
task <id> denotate "Waiting for reply"    # remove matching annotation
task <id> info                            # see all annotations
```

## Starting and Tracking

```bash
task <id> start     # mark as active (shows * indicator)
task <id> stop      # stop working on it
task active         # list currently active tasks
```

## Dependencies

```bash
task <id> modify depends:<other-id>     # <id> is blocked by <other-id>
task blocked                            # tasks waiting on dependencies
task blocking                           # tasks others depend on
task unblocked                          # tasks with no pending dependencies
```

## Recurring Tasks

```bash
task add recur:daily due:9am "Check email"
task add recur:weekly due:monday +work "Team standup"
task add recur:monthly due:1st "Pay rent"
task add recur:yearly due:2026-04-30 "Renew subscription"
task recurring                          # list all recurring tasks
```

When a recurring task is completed, Taskwarrior automatically creates the
next instance.

## Projects

```bash
task projects                           # list all project names
task project:Home list                  # tasks in a project
task summary                            # task counts by project
task project:Work.Frontend list         # subproject (dot notation)
```

## Tags

```bash
task tags                               # all tags in use
task +work list                         # tasks with tag
task -someday list                      # tasks WITHOUT tag
task <id> modify +urgent -later         # add and remove tags
```

## Reports

Taskwarrior has built-in reports. Useful ones:

| Report | Description |
|--------|-------------|
| `next` | Most urgent pending tasks |
| `ready` | Actionable (not waiting, not blocked) |
| `list` | All pending, most detail |
| `minimal` | Minimal columns |
| `long` | All columns |
| `all` | Every task regardless of status |
| `overdue` | Past due |
| `active` | Currently started |
| `blocked` | Blocked by dependencies |
| `blocking` | Others depend on these |
| `completed` | Done tasks |
| `waiting` | Hidden until wait date |
| `summary` | Status counts by project |
| `projects` | Project names only |
| `tags` | Tag names only |
| `burndown.weekly` | Graphical weekly burndown |
| `history.monthly` | Monthly history report |

```bash
task next
task ready
task long
task summary
task burndown.weekly
```

## Contexts

Contexts are saved filter + modification presets.

```bash
task context define work "project:Work or +work"
task context work               # activate work context
task context none               # clear context
task context list               # show defined contexts
task context show               # show active context
```

While a context is active, all commands are automatically filtered.

## Urgency Score

Taskwarrior ranks tasks by urgency (higher = more urgent). Factors:
due date, priority, age, tags, active status, dependencies.

```bash
task <id> info    # shows urgency score and all metadata
```

## Exporting Data

```bash
task export                         # export all tasks as JSON
task export project:Work            # filtered export
task export rc.json.array=on        # as a JSON array instead of lines
task completed export               # export completed tasks
```

## Attribute Reference

| Attribute | Description | Example |
|-----------|-------------|---------|
| `description` | Task text | `description:"Fix bug"` |
| `project` | Project name (dot for sub) | `project:Work.API` |
| `priority` | H / M / L | `priority:H` |
| `due` | Due date | `due:tomorrow` |
| `scheduled` | Scheduled start date | `scheduled:monday` |
| `wait` | Hidden until date | `wait:+3d` |
| `until` | Expiry (for recur) | `until:eoy` |
| `recur` | Recurrence | `recur:weekly` |
| `depends` | Blocking task IDs | `depends:5,7` |
| `status` | pending/completed/deleted/waiting | `status:waiting` |
| `entry` | Creation date | `entry.after:2026-01-01` |
| `end` | Completion/deletion date | `end:today` |
| `start` | Active start date | `start.any:` |
| `tags` | Comma list | `+work -someday` |
| `urgency` | Computed score | `urgency.above:5` |
| `uuid` | Unique ID | filter by exact ID |

## Attribute Modifiers

| Modifier | Example | Meaning |
|----------|---------|---------|
| _(none)_ | `due:today` | Fuzzy match |
| `.not` | `priority.not:L` | Not equal |
| `.before` / `.below` | `due.before:eow` | Strictly less than |
| `.after` / `.above` | `due.after:today` | Greater than or equal |
| `.none` | `project.none:` | Attribute is empty |
| `.any` | `start.any:` | Attribute is set |
| `.is` | `project.is:Home` | Exact match |
| `.isnt` | `project.isnt:Home` | Exact non-match |
| `.has` | `description.has:API` | Contains substring |
| `.word` | `description.word:bug` | Word boundary match |

## Common Workflows

### Morning review

```bash
task overdue                    # deal with overdue first
task due:today list             # what's due today
task ready                      # what can I start now
```

### Capture and triage

```bash
# Quick capture
task add "Review PR #42" project:Work +review due:today

# Triage inbox (tasks without project)
task project.none: list
task <id> modify project:Work priority:M
```

### Close out a task

```bash
task <id> info                  # confirm you have the right one
task <id> done
```

### Find and reschedule overdue tasks

```bash
task overdue                    # list them
task overdue modify due:tomorrow  # reschedule all (confirm first)
```

### Track active work

```bash
task <id> start
# ... work ...
task <id> done
```

## Tips

- Task IDs are local and change as tasks are completed. Use UUIDs for
  stable references in scripts.
- Short attribute abbreviations work if unambiguous: `pro:` for `project:`,
  `pri:` for `priority:`, `due:` stays `due:`.
- `task <filter> info` shows full details including annotations, urgency
  breakdown, and history.
- `task diagnostics` shows config paths and environment details.
- Use `task rc.color=off` to disable ANSI color for piped output.
