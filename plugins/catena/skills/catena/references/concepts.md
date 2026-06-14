# Catena Concepts & Data Model

Catena is an online-first, patch-based VCS. This file describes the mental model. For the commands
that act on this model, see [cli.md](cli.md). Catena is operated **only** through the `catena` CLI —
never call the server's REST API by hand.

## The big picture

```
working directory  ──diff──▶  Patch (a set of file Operations)
                                 │  maturity lifecycle
                                 ▼
        DraftLocal → DraftSynced → DraftShared → Proposed → Accepted
                                                              │ accept applies ops
                                                              ▼
                              Trunk  (file → content-hash state)
                                 └──snapshot──▶ Release (versioned)
```

- There is **no local commit graph**. The client compares the working directory against the
  server trunk every run, so there is no index/staging that can drift.
- **Trunk** = the canonical project state: a map of `file path → content hash`, produced by
  applying accepted patches in order. The list of accepted patches *is* the history.
- **Release** = an immutable named snapshot (e.g. `1.0.0`). A patch can target the trunk and/or
  one or more releases simultaneously.

## Patch

A patch is the atomic unit of change. Fields:

| Field | Meaning |
|-------|---------|
| `Id` | Server-assigned patch id. |
| `Author` | Who created it. |
| `ProjectId` | Owning project. |
| `Description` | Human summary (always set this). |
| `Maturity` | Lifecycle state (see below). Defaults to `DraftLocal`. |
| `Ops` | The list of file operations. |
| `Dependencies` | Patch ids that must be **Accepted** before this one can be accepted. |
| `Targets` | Where the patch applies. Defaults to `["trunk"]`; add release versions to also apply there. |
| `BaseHash` | Trunk hash the patch was authored against (basis for conflict detection). |
| `Timestamp`, `ProposedAt`, `AcceptedAt` | Timing metadata. |

## Maturity lifecycle

```
DraftLocal ──▶ DraftSynced ──▶ DraftShared ──▶ Proposed ──▶ Accepted
                    │                              ▲   │
                    └──────────────────────────────┘   │ (withdraw)
                       (skip DraftShared)               ▼
                                                   DraftSynced
```

**Exact allowed transitions** (anything else is rejected):

- `DraftLocal → DraftSynced`
- `DraftSynced → DraftShared`
- `DraftShared → Proposed`
- `DraftSynced → Proposed` (skip the shared step)
- `Proposed → Accepted`
- `Proposed → DraftSynced` (**withdraw** — go back to a draft to keep working)

Notes:
- `Accepted` is **terminal** — there is no transition out of it. To undo an accepted patch, create
  an **inverse patch** (revert), do not try to "un-accept".
- Setting maturity to `Accepted` requires the accept permission (Maintainer/Admin) and triggers the
  patch to be applied to the trunk and to every non-`trunk` target (release).
- A patch can only become `Accepted` once **all** its `Dependencies` are `Accepted`.

## Operations (`OpType`)

Each op acts on one file. `ContentBase64` carries the new content (base64 — binary-safe).

| OpType | Meaning | Uses `NewPath`? |
|--------|---------|-----------------|
| `Insert` | Add a new file (or insert content). | no |
| `Modify` | Change existing file content. | no |
| `Delete` | Remove a file. | no |
| `Rename` | Rename a file. | yes (`NewPath` = new name) |
| `Move`   | Move a file to a new path. | yes (`NewPath` = destination) |

`FileOperation` shape: `{ Type, File, ContentBase64?, NewPath? }`.

## Conflicts / overlaps

Catena detects when two patches touch overlapping regions/files (**overlaps**), surfaced via the
overlaps/conflicts endpoints at propose/review time. This is informational — there is no manual
three-way `git merge`. If patches conflict, resolve by rebasing your work (re-draft against the
current trunk) rather than force-merging. `BaseHash` records which trunk state a patch assumed.

## Reviews

Patches can carry reviews (approve/reject/comment) before acceptance, surfaced during the
`Proposed` stage.

## Roles & access

- Roles: `Contributor`, `Maintainer`, `Admin`.
- Projects can be **public** or private (`--public` at init / `isPublic` on create).
- Accepting patches, creating releases, and managing webhooks require Maintainer/Admin.
- User administration (`/users`) requires Admin.

## Workspace layout (`.catena/`)

- `.catena/config.json` — `{ "serverUrl", "projectId", "apiKey" }`.
- `.catena/ignore` — gitignore-style patterns (e.g. `bin/`, `obj/`, `node_modules/`) excluded from scans.
