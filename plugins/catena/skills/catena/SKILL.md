---
name: catena
description: Catena patch-based version control reference. ALWAYS use this skill when interacting with Catena — running the `catena` CLI, creating/proposing/accepting patches, working with the trunk, releases, reviews, or `.catena` workspaces. Catena is NOT Git: it is online-first and patch-based with a maturity lifecycle (DraftLocal → DraftSynced → DraftShared → Proposed → Accepted). Catena is operated ONLY through the `catena` CLI — never call the server's REST API by hand. Do not assume Git commands, branches, commits, or staging exist. Also trigger when the user mentions Catena, a `.catena/config.json`, patches with maturity, trunk state, or patch overlaps/conflicts.
---

# Catena Reference Skill

Catena is an **online-first, patch-based version control system** (built on .NET). It is **NOT Git**.
There are no commits, branches, staging area, or local history graph. The unit of change is a
**Patch** that moves through a **maturity lifecycle**, and the canonical state is the **trunk**
(a file → content-hash map built by applying accepted patches).

**Two hard rules:**
1. **Always operate Catena through the `catena` CLI.** The server has a REST API, but it is an
   internal implementation detail — **never call it by hand** (no curl, no manual HTTP). The CLI
   encapsulates correct workflow, base-hash/conflict handling, and safety.
2. **Never assume Git semantics.** Check the reference files before acting.

## When to read which file

| Task | Read |
|------|------|
| Running the `catena` CLI (init, draft, propose, accept, sync, log, diff, release, withdraw) | [references/cli.md](references/cli.md) |
| Understanding the model — patches, maturity, ops, trunk, releases, dependencies, conflicts | [references/concepts.md](references/concepts.md) |

## Critical differences from Git (read before doing anything)

- **No commits / branches / staging.** The unit of work is a **Patch**. A patch has a **maturity**: `DraftLocal → DraftSynced → DraftShared → Proposed → Accepted`. Only `Accepted` patches land in the trunk.
- **Online-first.** Almost every operation talks to a server (`serverUrl` in `.catena/config.json`). There is no offline commit history. The client recomputes the diff between the working directory and the trunk on every run — there is no local index/state to get out of sync.
- **Trunk replaces "main".** The trunk is not a branch; it is a `file → hash` state produced by applying accepted patches in order. History = the list of accepted patches.
- **Patches carry explicit `Dependencies` and `Targets`**, not a parent-commit DAG. A patch can target `trunk` and/or one or more **releases** at the same time. All dependencies must be `Accepted` before a patch can be accepted.
- **Releases are first-class snapshots**, not tags. Accepting a patch applies it to the trunk *and* to every non-`trunk` target.
- **`catena sync down` overwrites and DELETES local files** to match the trunk (like a hard checkout, including deletion of untracked files). This is destructive — see the warning in [references/cli.md](references/cli.md). Never run it casually.
- **Reverting is a last resort.** Undoing an accepted patch means creating an inverse patch. **Never revert autonomously** — always confirm with the user first; reserve it for extreme cases and use it with utmost care. The guiding rule of Catena is *never lose data or history.*
- **File content travels as base64** inside operations, so binary files are supported. Op types are `Insert, Modify, Delete, Rename, Move` — `Rename`/`Move` use the `NewPath` field.
- **Conflicts are detected server-side as patch overlaps**, surfaced at propose/review time (informational). There is no `git merge`/three-way merge step you drive manually.
- **Auth & roles:** the CLI handles auth automatically from `.catena/config.json` (you never send credentials by hand). Roles are `Contributor / Maintainer / Admin`; accepting a patch requires the accept permission (Maintainer/Admin).

## The normal workflow (happy path)

```
catena init --server <url> --new <project-name> --api-key <key>   # or --project <id> for existing
# ...edit files...
catena status                       # show what changed vs trunk
catena draft -m "what I changed" --sync   # create a patch, push it to the server
catena propose <patchId>            # submit for validation/review
catena accept  <patchId>            # (Maintainer/Admin) apply into trunk
```

Always pass a clear `-m` description. Prefer `--sync` on draft so the patch reaches the server.
Use `catena log` / `catena diff <patchId>` to inspect history before accepting.
