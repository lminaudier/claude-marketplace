---
name: lminaudier:mikado
description: Drive large-scale code changes (migrations, refactors, architectural shifts) using the Mikado Method — try, fail, record prerequisites as a dependency graph, revert, resolve leaves first, and visualize progress as an Excalidraw diagram.
argument-hint: [goal description, e.g. "migrate from Jest to Vitest"]
allowed-tools:
  - Bash
  - Read
  - Edit
  - Write
  - Glob
  - Grep
  - AskUserQuestion
---

Help the user drive a large-scale code change using the Mikado Method. The method's core loop: try a change, observe what breaks, record the failures as prerequisites in a dependency graph, revert, then resolve prerequisites bottom-up (leaves first). Visualize the graph as an Excalidraw diagram that the user can open at any time to see progress.

## Guardrails

- **Always revert after a failed experiment.** Never leave broken code in the working tree. Sequence: (1) stage and commit `.mikado/` changes, (2) `git checkout -- .` to discard everything else.
- **Never force-push.** The mikado branch accumulates commits linearly.
- **Never delete graph nodes.** Nodes transition status but are never removed from `graph.json`. If a prerequisite turns out to be unnecessary, mark it `"done"` with a note — do not delete it.
- **Always ask before attempting the root goal again.** The user may want to review state first.
- **Commit `.mikado/` changes separately** from functional code changes. This keeps graph metadata out of functional commits.
- **Never run destructive git commands** (`reset --hard`, `clean -fd`, `branch -D`) without explicit user confirmation.
- **Deterministic Excalidraw seeds.** Derive element seeds from node IDs (simple string hash) so regenerating the file produces stable diffs.
- **Preserve the user's working branch.** Create a dedicated branch and offer to merge back when complete.

---

## Resuming a previous session

Before initializing, check if `.mikado/graph.json` already exists. If it does:

1. Read it and present a summary: the goal, total node count, status breakdown (done / pending / in-progress / failed).
2. List the currently available leaf nodes (pending nodes whose children are all done).
3. Ask the user which leaf to work on next, or whether they want to attempt the root goal.

Skip to **Step 5** (pick a leaf) or **Step 2** (attempt the goal) based on the user's choice.

---

## Data model — `.mikado/graph.json`

The graph is stored as a single JSON file. Nodes encode their parent edges via `parentIds` — there is no separate edges array.

```json
{
  "version": 1,
  "goal": "migrate from Jest to Vitest",
  "verifyCommand": "npm test",
  "createdAt": "2026-04-27T10:00:00Z",
  "updatedAt": "2026-04-27T12:30:00Z",
  "nodes": [
    {
      "id": "root",
      "label": "Migrate from Jest to Vitest",
      "type": "goal",
      "status": "pending",
      "depth": 0,
      "parentIds": [],
      "errorOutput": null,
      "commitSha": null,
      "createdAt": "2026-04-27T10:00:00Z",
      "resolvedAt": null
    },
    {
      "id": "replace-jest-mock-auth",
      "label": "Replace jest.mock with vi.mock in auth module",
      "type": "prerequisite",
      "status": "pending",
      "depth": 1,
      "parentIds": ["root"],
      "errorOutput": "ReferenceError: jest is not defined at auth.test.ts:14",
      "commitSha": null,
      "createdAt": "2026-04-27T10:05:00Z",
      "resolvedAt": null
    }
  ]
}
```

### Node fields

| Field | Type | Description |
|-------|------|-------------|
| `id` | string | Short slug derived from the label (e.g. `replace-jest-mock-auth`). Human-readable, not a UUID. |
| `label` | string | Imperative verb phrase, max ~50 characters. |
| `type` | `"goal"` or `"prerequisite"` | The root node is `"goal"`, everything else is `"prerequisite"`. |
| `status` | `"pending"` / `"in-progress"` / `"done"` / `"failed"` | `"failed"` means the attempt revealed more sub-prerequisites — it transitions back to `"pending"` once those are recorded. |
| `depth` | number | Longest path from root to this node. Used for layout. Root is 0. |
| `parentIds` | string[] | IDs of the nodes this prerequisite enables. Array because one node can enable multiple parents (DAG, not tree). |
| `errorOutput` | string or null | Relevant error snippet from the failed attempt that created this prerequisite. |
| `commitSha` | string or null | SHA of the commit that resolved this node. |
| `createdAt` | ISO 8601 string | When the node was added to the graph. |
| `resolvedAt` | ISO 8601 string or null | When the node was marked `"done"`. |

### Invariants

- **No cycles.** Before adding an edge (child → parent), walk all descendants of the child via reversed `parentIds`. If the proposed parent is reachable, reject the edge and warn the user.
- **No duplicates.** Before adding a node, scan existing labels for semantic matches. If found, ask the user whether to reuse the existing node, add the new node as a child of the existing one, or create a separate node.
- **Depth consistency.** When adding a node, its depth = `max(depth of each parent) + 1`. If a node gains a new parent at a deeper level, update its depth and propagate to all descendants.

---

## Excalidraw rendering — `.mikado/mikado.excalidraw`

Generate this file from `graph.json` every time the graph changes. The user can open it in the Excalidraw VS Code extension or at excalidraw.com.

### File structure

```json
{
  "type": "excalidraw",
  "version": 2,
  "source": "mikado-method",
  "elements": [],
  "appState": {
    "gridSize": null,
    "viewBackgroundColor": "#ffffff"
  },
  "files": {}
}
```

### Color mapping

| Status | Background color | Meaning |
|--------|-----------------|---------|
| Goal (root) | `#d0ebff` (blue) | The target change |
| Pending | `#e9ecef` (grey) | Not yet attempted |
| In-progress | `#fff3bf` (yellow) | Currently being worked on |
| Done | `#d3f9d8` (green) | Resolved and committed |
| Failed | `#ffc9c9` (red) | Attempt failed, sub-prerequisites needed |

### Layout algorithm

1. **Group nodes by depth level.** Depth 0 is the root, depth 1 is immediate prerequisites, etc.
2. **Constants:** `NODE_W = 220`, `NODE_H = 80`, `H_GAP = 40`, `V_GAP = 120`.
3. **For each level**, calculate total width: `count * NODE_W + (count - 1) * H_GAP`.
4. **Center each level** horizontally. The leftmost node starts at `x = -(totalWidth / 2)`.
5. **Y position:** `depth * (NODE_H + V_GAP)`.
6. **Ordering within a level:** Sort nodes by the average x-center of their parents in the level above. This keeps related nodes grouped under their parent.
7. **Wide graphs:** If a level has more than 6 nodes, reduce `H_GAP` to 20 and `NODE_W` to 180. If more than 10 nodes, suggest the user group some prerequisites into an intermediate milestone node.

### Element templates

**Rectangle (one per node):**

```json
{
  "id": "<node-id>-rect",
  "type": "rectangle",
  "x": "<computed>",
  "y": "<computed>",
  "width": 220,
  "height": 80,
  "angle": 0,
  "strokeColor": "#1e1e1e",
  "backgroundColor": "<status-color>",
  "fillStyle": "solid",
  "strokeWidth": 2,
  "strokeStyle": "solid",
  "roughness": 0,
  "opacity": 100,
  "groupIds": [],
  "frameId": null,
  "roundness": { "type": 3 },
  "seed": "<deterministic-from-node-id>",
  "version": 1,
  "versionNonce": "<deterministic-from-node-id>",
  "isDeleted": false,
  "boundElements": [
    { "id": "<node-id>-text", "type": "text" }
  ],
  "updated": 1,
  "link": null,
  "locked": false
}
```

Add `{ "id": "<arrow-id>", "type": "arrow" }` to `boundElements` for each arrow that starts or ends at this rectangle.

**Text (one per node, bound to its rectangle):**

```json
{
  "id": "<node-id>-text",
  "type": "text",
  "x": "<rect-x + 10>",
  "y": "<rect-y + 10>",
  "width": 200,
  "height": 60,
  "angle": 0,
  "strokeColor": "#1e1e1e",
  "backgroundColor": "transparent",
  "fillStyle": "solid",
  "strokeWidth": 2,
  "strokeStyle": "solid",
  "roughness": 0,
  "opacity": 100,
  "groupIds": [],
  "frameId": null,
  "roundness": null,
  "seed": "<deterministic-from-node-id + 1>",
  "version": 1,
  "versionNonce": "<deterministic-from-node-id + 1>",
  "isDeleted": false,
  "boundElements": null,
  "updated": 1,
  "link": null,
  "locked": false,
  "text": "<node-label>",
  "fontSize": 16,
  "fontFamily": 1,
  "textAlign": "center",
  "verticalAlign": "middle",
  "containerId": "<node-id>-rect",
  "originalText": "<node-label>",
  "autoResize": true,
  "lineHeight": 1.25
}
```

**Arrow (one per edge, pointing from child up to parent):**

```json
{
  "id": "<child-id>-to-<parent-id>-arrow",
  "type": "arrow",
  "x": "<child-rect-center-x>",
  "y": "<child-rect-top-y>",
  "width": "<abs(dx)>",
  "height": "<abs(dy)>",
  "angle": 0,
  "strokeColor": "#1e1e1e",
  "backgroundColor": "transparent",
  "fillStyle": "solid",
  "strokeWidth": 2,
  "strokeStyle": "solid",
  "roughness": 0,
  "opacity": 100,
  "groupIds": [],
  "frameId": null,
  "roundness": { "type": 2 },
  "seed": "<deterministic-from-edge>",
  "version": 1,
  "versionNonce": "<deterministic-from-edge>",
  "isDeleted": false,
  "boundElements": null,
  "updated": 1,
  "link": null,
  "locked": false,
  "points": [[0, 0], ["<dx>", "<dy>"]],
  "startBinding": {
    "elementId": "<child-id>-rect",
    "focus": 0,
    "gap": 1
  },
  "endBinding": {
    "elementId": "<parent-id>-rect",
    "focus": 0,
    "gap": 1
  },
  "startArrowhead": null,
  "endArrowhead": "arrow"
}
```

Arrow coordinates: the arrow starts at the child's top-center `(x + NODE_W/2, y)` and ends at the parent's bottom-center `(parent_x + NODE_W/2, parent_y + NODE_H)`. The `points` array uses relative offsets from the arrow's origin `(x, y)`, so `dx = parent_center_x - child_center_x` and `dy = -(V_GAP + NODE_H)` (negative because parent is above).

### Seed generation

To produce deterministic seeds from node IDs, use a simple string hash:

```
seed = 0
for each character c in the string:
  seed = ((seed << 5) - seed) + charCode(c)
  seed = seed & 0x7fffffff  // keep positive 31-bit integer
```

Use the node ID for rectangle and text seeds (offset text seed by +1). Use `"<child-id>-<parent-id>"` for arrow seeds.

---

## Step 1 — Initialize

1. Ask the user for their goal if not provided as the argument.
2. Ask what command verifies success (e.g. `npm test`, `cargo build`, `make check`). This becomes the `verifyCommand`.
3. Create the `.mikado/` directory.
4. Create `graph.json` with a single root node: type `"goal"`, status `"pending"`, depth 0, empty parentIds.
5. Generate `mikado.excalidraw` — a single blue rectangle with the goal label.
6. Create a git branch: `mikado/<slugified-goal>` from the current HEAD.
7. Commit the `.mikado/` directory: `chore(mikado): initialize graph for "<goal>"`.
8. Tell the user the Excalidraw file is ready to open.

---

## Step 2 — Attempt the goal naively

1. Mark the root node as `"in-progress"`. Regenerate Excalidraw.
2. Tell the user you are about to attempt the goal. Describe what you plan to change.
3. Make the naive change.
4. Run the `verifyCommand`.
5. **If it succeeds on the first try:** mark the root as `"done"`, regenerate Excalidraw, commit, done. Congratulate the user.
6. **If it fails:** capture the error output (truncate to the most relevant ~50 lines), proceed to Step 3.

---

## Step 3 — Analyze failures

1. Read the error output from the failed attempt.
2. Identify distinct, independent prerequisites. For each one determine:
   - A short label (imperative verb phrase, max ~50 chars).
   - Which existing node(s) it enables (its `parentIds`).
   - The relevant error snippet.
3. Present the proposed prerequisites to the user as a numbered list. Ask them to confirm, modify, or add more before recording.

---

## Step 4 — Record prerequisites and revert

1. For each confirmed prerequisite:
   - Generate an ID: slugify the label (lowercase, hyphens, max 40 chars).
   - Check for duplicates: scan existing node labels. If a semantic match exists, ask the user whether to reuse it, make the new node a child of it, or create a separate node.
   - Check for cycles: from the new node, walk all descendants (nodes that list it in their `parentIds`, recursively). If the proposed parent is among them, reject and warn.
   - Add the node to `graph.json`.
2. Update the parent node's status back to `"pending"` (it cannot be resolved until its prerequisites are done).
3. Regenerate `mikado.excalidraw`.
4. **Revert all code changes** while preserving the graph:
   - `git add .mikado/` then `git commit -m "chore(mikado): record prerequisites for <parent-label>"`.
   - `git checkout -- .` to discard all other changes.
5. Proceed to Step 5.

---

## Step 5 — Pick a leaf node

**Edge direction recap:** In `graph.json`, `parentIds` on a node points *upward* — it lists the nodes this prerequisite enables. So if node A has `"root"` in its `parentIds`, A is a prerequisite of the root goal. The DAG flows from prerequisites (bottom) up to the goal (top).

A node's **sub-prerequisites** are nodes that list this node's ID in their own `parentIds`. These must be done before this node can be attempted.

**Definition:** A pending node B is a **workable leaf** if every node C where `B.id` is in `C.parentIds` has status `"done"`. In other words, all of B's sub-prerequisites are resolved. A node with no sub-prerequisites at all is trivially a leaf.

**Algorithm:** For each node B with status `"pending"`, collect all nodes C where `C.parentIds` includes `B.id`. If every such C has status `"done"`, B is a workable leaf.

If multiple leaves exist:
1. Present them to the user as a numbered list.
2. Suggest starting with the one that "unlocks" the most other nodes (count transitive dependents).
3. Let the user pick.

Mark the chosen leaf as `"in-progress"`. Regenerate Excalidraw. Proceed to Step 6.

---

## Step 6 — Attempt the leaf

1. Describe what needs to change to resolve this leaf.
2. Make the change.
3. Run the `verifyCommand`.
4. **If it succeeds:** proceed to Step 7.
5. **If it fails:** this leaf has its own sub-prerequisites. Capture the error output. Go back to Step 3 with this leaf as the parent for the new prerequisites. Set the leaf's status back to `"pending"`.

---

## Step 7 — Commit the resolved leaf

1. Stage the functional code changes (exclude `.mikado/`).
2. Commit with a conventional commit message: `refactor(<scope>): <leaf label>`.
3. Record the commit SHA on the node. Set status to `"done"` and `resolvedAt` to now.
4. Regenerate `mikado.excalidraw`.
5. Commit the updated `.mikado/` files: `chore(mikado): mark <leaf-label> as done`.
6. Proceed to Step 8.

---

## Step 8 — Repeat

1. Check if the root goal is now achievable: are all of the root's prerequisites done? (Find all nodes whose `parentIds` includes `"root"` — are they all `"done"`?)
2. **If yes:** ask the user if they want to attempt the root goal now. If confirmed, go to Step 2.
3. **If no:** go to Step 5 to pick the next leaf.
4. **When the root goal succeeds:** mark it `"done"`, regenerate the final Excalidraw diagram, and present a summary:
   - Total nodes resolved.
   - List of commits made (SHA + message).
   - The Excalidraw file path for the final green graph.
   - Offer to merge the mikado branch back into the original branch.

---

## Quick reference

| Action | Step |
|--------|------|
| Start a new Mikado goal | Step 1 |
| Resume an existing graph | See "Resuming a previous session" |
| Try the main goal | Step 2 |
| Record what broke | Steps 3–4 |
| Work on a leaf prerequisite | Steps 5–7 |
| Check if the goal is ready | Step 8 |
| View progress | Open `.mikado/mikado.excalidraw` |
