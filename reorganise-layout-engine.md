# Spec: the reorganise / layout engine (Phase 2)

**Status:** Draft, 2026-06-06. Scopes the work that turns Cat198x from a
verifier into a reorganiser. Companion to the binding decision
[`cat198x-asset-tooling.md`](../../decisions/cat198x-asset-tooling.md) and the
[`plan.md`](plan.md) rescue plan.

## Bottom line

Cat198x can **inventory and verify** today, but it cannot **reorganise into a
cohesive hierarchical structure** — the headline promise in the README. The
schema supports a hierarchy and `plan`/`apply` exist, but three pieces are
plumbed-but-not-wired:

1. **`dat add` never builds the hierarchy.** Every DAT lands as a single flat
   node — `create_node(…, &header.name, "dat", &header.name)` at
   `src/cli/dat.rs:149` and `:779`. The `manufacturer`/`system`/`category` node
   types exist only in tests. A live catalogue confirms it: all nodes are
   `node_type='dat'`, `path` is the bare collection name.
2. **`build_dest_path` is a stub** (`src/plan/generator.rs:232`): it returns
   `dest_root/rom_name` — flat, ignoring the game name, marked *"Phase 2 will
   add proper game folder structure."*
3. **Destinations are per-collection only**, set by exact name, and a collection
   with no `dest_path` is **silently skipped** during `plan`
   (`generator.rs:85`). There is no base/per-set destination, so reorganising a
   1,400-collection set means 1,400 manual `config set` calls.

This spec wires all three, smallest-useful-first.

## Key insight: derive the hierarchy from the canonical DAT path

The robust source of truth for a collection's place in the tree is **not** the
DAT name (TOSEC's `Manufacturer System - Category - Title` string is ambiguous to
split — "Acorn BBC" → `Acorn/BBC`, but "Nintendo Famicom & Entertainment System"
does not split cleanly). It is the **DAT file's path relative to the set root**.

`DatRoot/` already stores DATs nested in exactly the target layout:

```
DatRoot/TOSEC-PIX/Acorn/BBC/Magazines/Laserbug/Acorn BBC - Magazines - Laserbug (...).dat
                  └── relative path IS the hierarchy ──┘
```

So when `dat add --recursive <set-root>` walks the tree, the relative directory
path of each DAT is the manufacturer/system/category chain. No name parsing, no
lookup table, and it tracks TOSEC's own conventions because it *is* TOSEC's
folder layout. This makes the canonical-DAT-location decision (move DATs into the
nested `DatRoot/` and register from there) a **prerequisite** of clean hierarchy
injection, not a separate chore. The flat Downloads pack cannot drive this; the
nested `DatRoot/` can.

Fallback for non-recursive `dat add` of a single flat DAT: parse the name as
today (one flat node). Hierarchy is a recursive-add feature.

## The three changes

### A. Hierarchy injection at recursive `dat add`  *(landed 2026-06-06)*

`add_dats_recursive` has each DAT's full path and the recursion root, so it
computes `rel = dat_path.parent().strip_prefix(root)` and **stores that
`/`-joined relative path on the collection's single `dat` node** (`path` field).
A single-file add, or a DAT sitting at the add root, falls back to the flat
collection name — today's behaviour.

**Simplification from the original sketch (no intermediate node chain).** The
sketch proposed building a `manufacturer`/`system`/`category` node *chain*.
Dropped, deliberately: `dat_nodes` is scoped to one `collection_version`
(`version_id` FK), so intermediate nodes cannot be *shared* across collections —
each DAT would get its own unshared linear branch with no consumer. The
destination builder (§B) needs only the leaf's relative `path`, which is now
populated. Grouping/navigation by manufacturer/system (see `Q3`) will match on
the path string until a consumer justifies a shared tree, which would need a
schema change, not just node rows. YAGNI until then.

- **Acceptance (met):** after a recursive add over a nested tree, each
  collection's node `path` is its directory relative to the add root
  (`Acorn/BBC/Magazines/Laserbug`); a flat or single add keeps the collection
  name. Unit-tested in `cli::dat::tests`.

### B. Hierarchical `build_dest_path`

- Signature gains the collection's node `path` (the relative hierarchy).
- Compose: `<base>/<hierarchy>/<title>/<rom_name>`, where `title` is the leaf
  collection and the `<title>` folder is omitted for single-ROM loose games when
  `output_format = loose` (the TOSEC default) to avoid a redundant nesting level.
- Respect `output_format` (loose vs zip/7z) and `merge_mode` as already modelled;
  start with `loose`, which covers TOSEC/PIX/ISO.
- **Acceptance:** a dry-run plan for a known collection emits destination paths
  byte-identical to its real on-disk canonical paths; already-correct files
  produce zero operations.

### C. Base / per-set destination

- Add a destination that a whole set inherits, so collections need no per-name
  config. Two options, prefer the first:
  - **Per-set base** keyed on the set root node: `config set <set> base_dest
    <path>`; a collection's destination resolves to `base_dest + relative
    hierarchy`.
  - **Global default** in `config.toml` (`default_dest_path`), same resolution.
- Resolution order in `generator.rs`: explicit per-collection `dest_path` →
  per-set `base_dest` + hierarchy → global default + hierarchy → **skip with a
  reported warning** (fixes the silent-skip; print "N collections skipped: no
  destination resolved").
- **Acceptance:** `config set TOSEC-PIX base_dest /Volumes/Data/TOSEC-PIX`
  followed by `plan` covers every PIX collection; the summary names any skipped.

## In-place tidy falls out for free

Set each set's `base_dest` to its existing root (`/Volumes/Data/TOSEC-PIX`, …).
The derived hierarchy reproduces the current layout, so correctly-placed files
no-op and only misnamed/misplaced/duplicated files generate move/rename
operations. "Tidy in place" needs no new mode — it is `base_dest = set root`.

## Out of scope (for this phase)

- Repack/format conversion beyond what `apply` already does.
- MAME merge-mode reorg nuances (parent/clone) — verify first, reorg later.
- Extracting source material toward `reference/` (the PIX manuals/magazines
  routing in `plan.md`) — a later phase that builds on this one.

## Open questions

1. **Completing the nested `DatRoot/`.** It is currently a partial copy (e.g.
   1,103 PIX vs the pack's 1,332). Populating the missing DATs into the right
   nested folders needs the same hierarchy mapping — either a one-off sort of the
   flat pack into `DatRoot/`, or accept name-parsing for the gap. Decide before
   the canonical rebuild.
2. **Free-space guard.** DATs describe ~2.6 TB against ~433 GB free; `plan`/`apply`
   should pre-check destination capacity and refuse/​warn. Small, worth folding in
   here.
3. **Grouped status/stats** (roll up "all TOSEC-PIX", per-system) for analysis —
   adjacent, not blocking; track separately.
