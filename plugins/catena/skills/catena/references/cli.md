# Catena CLI Reference

The `catena` CLI is the primary way to use Catena. Every command except `init` requires an existing
workspace (`.catena/config.json`); run `catena init` first.

## Global

- `--path <dir>` — operate on a workspace directory other than the current one (works on any command).
- `catena --help` / `catena -h` — usage.

## Commands

### `init` — create / connect a workspace
Writes `.catena/config.json`. Either create a new project or connect to an existing one.

```
catena init --server <url> --new <project-name> --api-key <key> [--public] [--path <dir>]
catena init --server <url> --project <existing-project-id> --api-key <key> [--path <dir>]
```
- `--server <url>` — server URL (default `http://localhost:5000`).
- `--new <name>` — create a new project on the server.
- `--project <id>` — connect to an existing project (mutually exclusive with `--new`).
- `--api-key <key>` — API key used for the `X-Api-Key` header on all later calls.
- `--public` — make the new project public.

### `status` — what changed vs trunk
```
catena status
```
Shows files added/modified/deleted in the working directory relative to the current trunk. Read-only.

### `draft` — create a patch from working-dir changes
```
catena draft -m "description" [--sync] [--depends-on <patchId>]... [--target <id>]...
```
- `-m <message>` — patch description (always provide one).
- `--sync` — push the draft to the server (moves it toward `DraftSynced`). Without `--sync` it stays local.
- `--depends-on <patchId>` — declare a dependency; repeatable.
- `--target <id>` — apply target in addition to `trunk` (e.g. a release version); repeatable.

### `propose` — submit a patch for validation/review
```
catena propose <patchId>
```
Moves the patch to `Proposed`. Conflicts/overlaps against the current trunk surface here.

### `accept` — apply a patch into the trunk (Maintainer/Admin)
```
catena accept <patchId>
```
Moves `Proposed → Accepted` and applies the ops to the trunk and to every non-`trunk` target.
Requires accept permission and all dependencies already `Accepted`. **Irreversible** except via revert.

### `withdraw` — pull a proposed patch back to draft
```
catena withdraw <patchId>
```
Moves `Proposed → DraftSynced` so you can keep editing.

### `log` — patch history
```
catena log [--file <path>] [--author <name>]
```
Lists accepted patches; filter by file or author.

### `diff` — show a patch's changes
```
catena diff <patchId>
```

### `release` — create a release snapshot
```
catena release <version>
```
Creates an immutable named snapshot of the current trunk (e.g. `catena release 1.2.0`).

### `sync` — reconcile the working directory with the trunk
```
catena sync up    [-m "description"]        # push local changes as a patch
catena sync down  [--patch <patchId>]       # pull trunk into the working directory
```
- `up` — upload local changes.
- `down` — download the trunk into the working directory. Optionally overlay a specific patch with `--patch <id>`.

> ⚠️ **`catena sync down` is destructive.** It overwrites locally modified files and **deletes**
> local files that are not present in the target (including brand-new, never-drafted files), to make
> the working directory match the trunk. Make sure local work is drafted/synced first. Do not run it
> to "refresh" if you have uncommitted work you care about.

### `revert` — create an inverse patch (last resort, confirm first)
Reverting an accepted patch creates an **inverse patch** (the `Accepted` state is terminal and
cannot be rolled back in place).

> ⚠️ **Never revert autonomously.** Always confirm with the user first. Reverting is reserved for
> extreme cases and must be done with utmost care, or only when the user explicitly asks for it.
> Catena's guiding rule is *never lose data or history* — an unprompted revert violates it.

## Typical session

```bash
catena init --server https://catena.example.app --new my-app --api-key $CATENA_KEY
# edit files...
catena status
catena draft -m "Add login form" --sync
catena propose <patchId>
# reviewer / maintainer:
catena accept <patchId>
catena release 0.1.0
```
