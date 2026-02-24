---
name: bear
description: Interact with Bear notes using the bcli CLI. Use this skill whenever the user mentions Bear notes, wants to find, read, create, edit, organize, or manage their notes or todos. Activates on requests like "find my note about X", "create a note for Y", "what todos do I have", "add this to my notes", etc.
allowed-tools:
  - Bash
---

# Bear Notes Integration

This skill uses `bcli` (better-bear-cli) to interact with Bear notes. All read operations must be preceded by a sync. Writes take effect immediately in Bear.

## Guardrails

- **Always sync before reading.** Run `bcli sync` before any `ls`, `get`, `search`, `tags`, or `todo` command.
- **Never empty the trash.** Do not run any command that empties the Bear trash. If the user asks, refuse and explain why.
- **Highlight trashed notes.** Trashing does not require confirmation, but always clearly surface which note(s) were trashed at the end of your response so the user can recover them if needed.
- Prefer `--json` output when parsing results programmatically (IDs, iteration).
- When creating or editing notes, show the user the final content before writing if it is longer than a few lines.

## Workflow

### Before any read operation

Always sync first:

```bash
bcli sync
```

If the user has not authenticated yet, `bcli auth` will open browser-based Apple Sign-In.

### Finding notes

**Full-text search** (searches title, tags, and body):
```bash
bcli search "query"
```

**List by tag:**
```bash
bcli ls --tag "tagname"
```

**List all notes:**
```bash
bcli ls --all
```

**Browse tag hierarchy:**
```bash
bcli tags
```

Use `--json` when you need to extract IDs or iterate over results:
```bash
bcli search "query" --json
bcli ls --tag "tagname" --json
```

### Reading a note

```bash
bcli get <id>           # full details
bcli get <id> --raw     # markdown body only
```

### Creating a note

```bash
bcli create "Title" -b "Body content" -t "tag1,tag2"
```

For multi-line bodies, use stdin:
```bash
bcli create "Title" -t "tag" --stdin <<'EOF'
Body content here.
EOF
```

### Editing a note

**Append to existing content:**
```bash
bcli edit <id> --append -b "New content to add"
```

**Replace via stdin (full rewrite):**
```bash
bcli edit <id> --stdin <<'EOF'
Full new content.
EOF
```

**Open in editor interactively:**
```bash
bcli edit <id> --editor
```

### Todos

**List all notes with incomplete tasks:**
```bash
bcli todo
```

**List todos in a specific note:**
```bash
bcli todo <id>
```

**Toggle a specific task (1-based index):**
```bash
bcli todo <id> --toggle 2
```

### Trashing a note

```bash
bcli trash <id>
```

No confirmation needed, but **always explicitly tell the user which note was trashed** (title + ID) at the end of your response so they can find and restore it from the Bear trash if needed.

### Exporting notes

```bash
bcli export ./output-dir
bcli export ./output-dir --tag "tagname"
bcli export ./output-dir --frontmatter
```

## Common Patterns

**Find a note and read it:**
```bash
bcli sync
bcli search "query" --json   # get ID from results
bcli get <id> --raw
```

**Search across tags, then filter:**
```bash
bcli sync
bcli ls --tag "project" --json
```

**Add a quick note:**
```bash
bcli create "Quick Note" -b "Content" -t "inbox"
```

**Capture today's date in a note title:**
```bash
bcli create "$(date +%Y-%m-%d) - Meeting Notes" -t "meetings" --stdin <<'EOF'
...
EOF
```

**Mark a todo done:**
```bash
bcli sync
bcli todo <id> --json        # find task index
bcli todo <id> --toggle <n>
```
