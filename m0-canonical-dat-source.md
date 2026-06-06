# M0: canonical DAT source — decision and procedure

**Status:** Decided 2026-06-06; the on-Capsule file work is queued behind the
running scan. Companion to [`reorganise-layout-engine.md`](reorganise-layout-engine.md)
and [`roadmap.md`](roadmap.md).

## Decision

The canonical DAT source is the **nested `DatRoot/` tree on the Time Capsule**,
and collections are registered with **recursive `dat add` from the `DatRoot`
parent**, so each collection's node path carries its set *and* hierarchy:

```
dat add -r /Volumes/Data/DatRoot
  → node path "TOSEC-PIX/Acorn/BBC/Magazines/Laserbug"
config set-default dest_path /Volumes/Data
  → destination "/Volumes/Data/TOSEC-PIX/Acorn/BBC/Magazines/Laserbug"
```

That destination is exactly where the files already live, so "tidy in place" is
just `default_dest_path = /Volumes/Data` with no per-collection configuration.

This was settled over the two alternatives:

- **Name-parsing in `dat add`** (derive the hierarchy from the collection name
  like `Acorn BBC - Magazines - Laserbug`). Rejected as the primary route: the
  publisher/system boundary inside the first segment (`Acorn BBC` →
  `Acorn/BBC`) has no delimiter and needs a publisher lookup — fragile, and it
  bakes that fragility permanently into the tool.
- **A one-off sort of the flat Downloads pack into folders.** This is what
  *produces* the nested tree, but the sorting logic is the same fragile mapping;
  doing it once into `DatRoot` (reviewable) is acceptable, doing it on every add
  is not. Recorded so the "no ad-hoc TOSEC scripts" rule is respected: if the
  sort needs real logic it becomes a `cat198x` subcommand, not a loose script.

The engine was built (M1) to derive the hierarchy from the DAT's path, which is
why the nested `DatRoot` is the source of truth rather than the flat pack.

## The open question: is `DatRoot` complete and aligned?

`DatRoot/TOSEC-PIX` held ~1,103 DATs against the v2025-03-13 pack's 1,332. Two
things must hold before the canonical rebuild (M4):

1. **Coverage** — `DatRoot` must contain a DAT for every collection present on
   disk under `/Volumes/Data/<set>/`. (Missing pack DATs that correspond to
   nothing on disk don't matter; missing ones that *do* have files on disk would
   read as uncatalogued.)
2. **Alignment** — each DAT's path under `DatRoot/<set>/` must mirror the file
   layout under `/Volumes/Data/<set>/`, so `default_dest_path + node path` equals
   the files' current location and "in place" no-ops instead of moving everything.

Both are cheap to verify but need the AFP volume quiet, so they run **after the
scan**, not during it (the scan saturates the mount).

## Post-scan procedure (queued; runs with M4)

Order matters — `Q1` (idempotent add) and `dat relink` (`Q4`) are in place to make
this clean and repeatable.

1. **Verify coverage & alignment** (read-only):
   - Per set, compare the set of `DatRoot/<set>/**.dat` relative paths against the
     on-disk `/Volumes/Data/<set>/` collection folders. List any on-disk
     collection with no matching DAT.
   - If gaps exist, complete `DatRoot` from the flat pack (the one-off sort), and
     re-verify. This is the only step that might need the name→folder mapping.
2. **Rebuild registrations onto canonical paths.** The current catalogue points
   at `~/Downloads/...`; re-point it at `DatRoot`. Either `dat relink
   /Volumes/Data/DatRoot` (same-named files, no re-parse) for the existing
   registrations, or remove + `dat add -r /Volumes/Data/DatRoot` for a clean
   rebuild that also populates the node hierarchy (idempotent, so re-runnable).
   The scanned file inventory is untouched either way.
3. **Set the destination:** `cat198x config set-default dest_path /Volumes/Data`.
4. **Plan + dry-run:** `cat198x plan` then `cat198x apply --dry-run`, and review.
   For a correctly-aligned `DatRoot`, a correctly-placed, correctly-named file is
   "already correct" and generates no operation; only misnamed/misplaced/
   duplicate files produce moves. A large move count is the signal that alignment
   (step 1) is off — investigate before applying.

## Why it's safe to wait

Nothing on disk changes until step 4's reviewed `apply` (a separate, explicit
step). The engine is complete and tested; M0/M4 are an operational sequence over
real data, gated only on the scan finishing so the verification and any sort
aren't fighting the scan for the mount.
