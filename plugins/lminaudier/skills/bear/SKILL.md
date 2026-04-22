---
name: bear
description: >-
  Interact with the Bear note-taking app using the bearcli CLI.
  Use this skill whenever the user mentions Bear, Bear notes, or wants to
  read, search, create, edit, tag, pin, archive, trash, or manage notes
  and attachments in Bear. Activates on requests like "find my Bear note
  about X", "create a note in Bear", "add a tag to my note", "search Bear
  for meeting notes", "append to my daily log", "list my Bear tags", etc.
allowed-tools:
  - Bash
---

# bearcli -- Bear Notes CLI

bearcli is a command-line tool for reading and writing Bear notes. It
operates directly on the local Bear database -- no network calls, no
telemetry.

## Guardrails

- **Read before write**: always `cat` or `show` a note before overwriting
  with `write`. Use `--base <hash>` for optimistic concurrency when
  overwriting.
- **No blind destructive ops**: never trash or delete attachments without
  first showing the note to the user. Confirm before trashing.
- **Preserve attachments on write**: when using `write` to overwrite a
  note, attachment references (inline markdown links) must be present in
  the new content or the attachments are removed from the note.
- **Encrypted notes**: encrypted note content is not accessible through
  the CLI. Metadata is still available.
- **Use `--format json`** for structured parsing in pipelines. Default
  TSV has no header.

## Quick Reference

| Task | Command |
|------|---------|
| Read note content | `bearcli cat <id>` |
| Show note metadata | `bearcli show <id> --format json --fields all` |
| List all notes | `bearcli list --format json` |
| Search notes | `bearcli search "query" --format json` |
| Find in a note | `bearcli search-in <id> --string "text"` |
| Create a note | `bearcli create "Title" --content "Body"` |
| Edit (find & replace) | `bearcli edit <id> --at "old" --replace "new"` |
| Append to note | `bearcli append <id> --content "text"` |
| Overwrite note | `bearcli write <id> --content "full content" --base <hash>` |
| List tags | `bearcli tags list` |
| Add tags | `bearcli tags add <id> work draft` |
| Pin a note | `bearcli pin add <id> global` |
| Trash a note | `bearcli trash <id>` |
| Archive a note | `bearcli archive <id>` |
| Restore a note | `bearcli restore <id>` |
| Open in Bear | `bearcli open <id>` |
| List attachments | `bearcli attachments list <id>` |

## Command Discovery

bearcli is self-documenting:

```bash
bearcli -h                    # Short overview
bearcli help                  # Full reference for every command
bearcli help <subcommand>     # Detailed help for one command
```

## Note Identification

Notes can be identified by ID or by title (case-insensitive):

```bash
bearcli cat <note-id>
bearcli cat --title "My Note"
```

`create` returns the new note's ID -- capture it for subsequent commands.

## Output Formats

| Format | Flag | Notes |
|--------|------|-------|
| TSV | `--format tsv` | Default. No header. Fields use `\n \t \\` escaping. |
| CSV | `--format csv` | RFC 4180 with header row. |
| JSON | `--format json` | JSON Lines (one object per line). Best for parsing. |

Use `--fields f1,f2` to select fields, or `--fields all` for everything.
Content is excluded from `all` -- use `--fields all,content` to include it.

## Reading Notes

### `cat` -- raw content

```bash
bearcli cat <id>
bearcli cat --title "Mars"
bearcli cat <id> --offset 0 --limit 500    # byte-range slice
```

### `show` -- structured metadata

```bash
bearcli show <id> --format json --fields all
bearcli show --title "Mars" --fields id,title,tags,hash,modified
```

Default fields: `id, title, tags`.
All fields: `id, title, locked, tags, hash, length, created, modified,
pins, location, todos, done, attachments, content`.

## Searching Notes

### `search` -- Bear search syntax

```bash
bearcli search "meeting notes"
bearcli search "@today @todo" --format json
bearcli search "#work @last7days" -n 10
bearcli search "@todo" --count
```

**Search operators:**

| Operator | Meaning |
|----------|---------|
| `keyword` | Full-text search |
| `"exact phrase"` | Exact match |
| `-term` | Exclude term |
| `#tag` | Has tag |
| `!#tag` | Exact tag (no children) |
| `#*/tag` | Subtags only |
| `@today` / `@yesterday` | Modified today/yesterday |
| `@lastXdays` | Modified in last X days (e.g. `@last7days`) |
| `@date(YYYY-MM-DD)` | Modified on specific date |
| `@date(<date)` / `@date(>date)` | Modified before/after date |
| `@ctoday` / `@createdXdays` | Created today / in last X days |
| `@todo` | Has incomplete tasks |
| `@done` | All tasks complete |
| `@task` | Has any task |
| `@tagged` / `@untagged` | Has/lacks tags |
| `@title` | Restrict text search to titles |
| `@pinned` | Globally pinned notes |
| `@images` / `@files` / `@attachments` | Has media |
| `@code` | Contains code blocks |
| `@locked` | Encrypted notes |

Combine freely: `bearcli search "@today @todo meeting #work"`.

### `search-in` -- find within a single note

```bash
bearcli search-in <id> --string "TODO"
bearcli search-in --title "Mars" --string "water" --format json
bearcli search-in <id> --string "TODO" --count
```

Returns offset and context snippet for each match.

## Listing Notes

```bash
bearcli list
bearcli list --tag work
bearcli list --tag work --sort modified:asc --fields id,title,modified
bearcli list -n 20 --format json --fields all
bearcli list --location archive
bearcli list --location trash
bearcli list --count
```

Sort fields: `pinned, modified, created, title` (append `:asc` or `:desc`).
Location filter: `notes` (default), `trash`, `archive`, `all`.

## Creating Notes

```bash
bearcli create "My Note" --content "Body text"
bearcli create --content "# Quick Capture\nSome thoughts"
bearcli create "My Note" --tags "work,draft" --format json
printf "line1\nline2" | bearcli create "My Note"
bearcli create "My Note" --if-not-exists    # idempotent
```

- Title is optional -- Bear derives it from the first `#` heading.
- `--tags` inserts tags at the position configured in Bear settings.
- `--if-not-exists` returns the existing note if one with the same title
  exists.
- `create` returns a structured row (id, title, tags) -- capture the ID.

## Editing Notes

### `edit` -- find and replace / insert

```bash
bearcli edit <id> --at "TODO" --replace "DONE"
bearcli edit <id> --at "## Notes" --insert "\nNew line after heading"
bearcli edit <id> --at "cat" --replace "dog" --all --word
bearcli edit <id> --at "old" --replace "new" --ignore-case
```

- `--at` + `--replace`: replace matched text.
- `--at` + `--insert`: insert text after the match.
- `--all`: apply to all occurrences.
- `--word`: match whole words only.
- Repeat `--at`/`--replace`/`--insert` for batch edits.
- Text arguments interpret `\n`, `\t`, `\r`, `\\`.

### `write` -- overwrite entire content

```bash
# Always read hash first for safety
bearcli show <id> --fields hash
bearcli write <id> --base <hash> --content "# Title\nNew body"
printf "# Title\nBody" | bearcli write <id> --base <hash>
```

- `--base <hash>` rejects the write if the note changed since the hash
  was obtained (optimistic concurrency). Always use this.
- Content replaces everything. Preserve the `#` heading (title) and
  attachment references.

### `append` -- add content

```bash
bearcli append <id> --content "New paragraph"
bearcli append --title "Mars" --content "Update" --position beginning
printf "New content" | bearcli append <id>
```

- `--position end` (default): appends before bottom-placed tags.
- `--position beginning`: inserts after title and top-placed tags.

## Tags

```bash
bearcli tags list                          # all tags in Bear
bearcli tags list <id>                     # tags on a specific note
bearcli tags add <id> work "work/meetings" # add tags to note
bearcli tags remove <id> draft wip         # remove tags from note
bearcli tags rename work job               # rename across all notes
bearcli tags rename old new --force        # merge into existing tag
bearcli tags delete draft                  # delete from all notes
```

Tag format: `#single`, `#multi word#`, `#nested/child`.
Input with or without `#` -- surrounding `#` and whitespace are stripped.

## Pins

```bash
bearcli pin list                    # every pin context in use
bearcli pin list <id>               # pins on a single note
bearcli pin add <id> global         # pin in All Notes
bearcli pin add <id> work projects  # pin within tag views
bearcli pin remove <id> global
```

## Note Lifecycle

```bash
bearcli trash <id>                  # soft-delete (recoverable)
bearcli archive <id>                # hide from active list
bearcli restore <id>                # move back to active notes
bearcli open <id>                   # open in Bear app
bearcli open <id> --edit            # open with cursor in editor
bearcli open <id> --new-window      # open in new window
bearcli open <id> --float           # open in floating panel
```

## Attachments

```bash
bearcli attachments list <id>
bearcli attachments list --title "Mars" --format json
cat photo.jpg | bearcli attachments add <id> --filename photo.jpg
bearcli attachments save <id> --filename photo.jpg > photo.jpg
bearcli attachments delete <id> --filename photo.jpg
```

## Common Agent Workflows

### Read -> Edit -> Verify

```bash
# 1. Read current content and hash
bearcli cat <id>
bearcli show <id> --fields hash

# 2. Make targeted edits
bearcli edit <id> --at "old text" --replace "new text"

# 3. Verify
bearcli cat <id>
```

### Read -> Rewrite (full overwrite)

```bash
# 1. Read content and hash
CONTENT=$(bearcli cat <id>)
HASH=$(bearcli show <id> --format json --fields hash | jq -r .hash)

# 2. Modify content externally, then write back
bearcli write <id> --base "$HASH" --content "# Title\nNew content"
```

### Create and organize

```bash
# Create note, capture ID
ID=$(bearcli create "Meeting Notes" --content "Agenda..." --tags "work,meetings" --format json | jq -r .id)

# Pin it
bearcli pin add "$ID" global

# Later, archive it
bearcli archive "$ID"
```

### Daily log / journal pattern

```bash
# Append to an existing daily note (idempotent create)
ID=$(bearcli create "Journal 2026-04-22" --if-not-exists --format json | jq -r .id)
bearcli append "$ID" --content "\n## 14:30 -- Status update\nAll good."
```

### Search and process results

```bash
# Find all notes with incomplete tasks tagged #work
bearcli search "@todo #work" --format json --fields id,title,todos,done

# Count notes modified today
bearcli search "@today" --count
```

## Conventions

| Convention | Detail |
|------------|--------|
| Exit codes | 0 success, 1 business error, 64 usage error |
| Mutating commands | No output on success (except `create`) |
| Timestamps | ISO 8601 UTC (`YYYY-MM-DDTHH:MM:SSZ`) |
| Text escaping | `\n`, `\t`, `\r`, `\\` in flag values; stdin is not unescaped |
| `--content` omitted | Reads from stdin |
| `--no-update-modified` | Available on `edit`, `write`, `append`, `attachments add/delete` |
