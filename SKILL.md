---
name: "disk-scanner-carlos"
description: "Organize files by MOVING (not copying) to folders, group into fixed 2-year blocks by default (stretching to absorb small trailing years, never leaving a near-empty standalone year folder), verify moves proportional to real risk, log every operation incrementally to a single DISK_OPERATIONS log at the drive root. Cross-disk migrations build a same-device staging folder only — the sandbox cannot delete files on mounted external drives, so true cross-disk moves must be finished by the user physically. NEVER deletes anything."
---

## Golden Rule

🔴 **NO AUTO-EXECUTION | ALWAYS MOVE, NEVER COPY | NEVER DELETE ANYTHING**

Every operation shows preview first. Nothing happens without explicit user authorization.
All file operations use **mv** (MOVE), not cp (copy). This skill NEVER deletes files or folders — not originals, not duplicates, not empty folders. Everything that needs to go somewhere gets MOVED, never removed.

Flow: Show preview → Ask confirmation → Wait for yes/no → Execute only if authorized → **Verify move (see Verification Tiers)** → **Log incrementally**

## ⚠️ HARD ENVIRONMENT CONSTRAINT: cannot delete files on mounted external drives

Confirmed by direct test: the sandbox this skill runs in can CREATE and WRITE files on connected/mounted external drives, but CANNOT delete/unlink files on them via any shell tool available (`rm`, `os.unlink`, `shutil.move`'s fallback delete step all fail with `PermissionError: Operation not permitted`). This is a hard sandbox restriction, not a skill policy choice.

**Consequence for cross-device operations:**
- A move WITHIN the same disk/filesystem (same `st_dev`) works fine — it's an atomic rename, no delete step involved. Use this freely (Tier 1).
- A move ACROSS two different disks (e.g. from `WD2TB BACKP` to `1TB FAM 24`) requires copy-then-delete-original. The copy will succeed; **the delete-original step will fail every time.** Do NOT attempt this directly — it silently leaves a verified-good copy at the destination but ALSO leaves the original in place, becoming an unintended duplicate that nothing flags automatically unless you check for it.
- **Never call `shutil.move`/`mv` with source and destination on two different mounted drives.** Check `os.stat(src).st_dev != os.stat(dest).st_dev` first — if they differ, this is cross-device and cannot be completed as a real move by this skill.

**What to do instead for multi-drive migrations:**
1. Build a staging folder for each target drive INSIDE the source drive itself (e.g. `<SOURCE_DRIVE>/MOVER_A_<TARGET_DRIVE_NAME>/`), organized exactly as it should land on the target. This is a same-device move — fast, safe, fully completable by this skill.
2. Tell the user clearly that the skill cannot physically move data between two separate mounted drives, and that they need to drag/copy the staging folder to the real target drive themselves (Finder, Explorer, etc.), then the skill can help verify/clean up afterward if asked.
3. If the user says they already relocated a staging folder to its real drive (this happens mid-job on large transfers), **verify by checking file counts on the real target drive** before continuing or reconciling logs — don't assume the local staging path still exists, and don't try to re-verify by re-hashing across drives unless asked.
4. If a direct cross-device move was attempted before this was known and left a duplicate (copy succeeded, original delete failed), log it as an incident (see Log section), tell the user exactly which file(s) and both locations, and let them decide whether to remove the duplicate themselves — this skill cannot delete it from either drive on its own once it needed a cross-device delete.

## Communication style — concise, precise, listed

Every preview, clarification, question, or result summary shown to the user must be:
- **Concise**: no filler sentences, no restating context the user already knows.
- **Precise**: exact counts, exact folder names, exact numbers — never vague language like "several" or "many."
- **Listed**: use a bullet or numbered list for anything with more than one item (folders, counts, questions, options). Never bury multiple facts in a paragraph.

Apply this to every step of the Safety Protocol below (PREVIEW, EXPLAIN, ASK, REPORT) — not just final summaries.

## Priority #1: Anti-Duplication Verification — proportional to real risk

**Verification Tiers (pick per move, based on `os.stat(src).st_dev` vs `os.stat(dest_parent).st_dev`):**

- **Tier 1 — Same device (same disk/filesystem):** the only tier this skill can safely execute end-to-end as a true move. After the move: confirm the destination file exists, confirm the source path no longer exists, confirm destination size matches the size recorded before the move. Do NOT hash full file contents — wasted I/O for an atomic rename.
- **Tier 2 — Cross device (different physical disk/mount):** per the hard constraint above, this skill CANNOT complete a true move here (the delete-original step will fail). Do not attempt it as a move. If the user explicitly wants a copy left on both drives temporarily, that's a copy operation, not a move — be explicit that the original will remain on the source drive and this is not what "move" normally means in this skill's context.

**Mandatory sequence for Tier 1 (the only tier actually executed as MOVE):**
1. Before moving: record source size.
2. Execute the move (`mv` / `shutil.move`, which uses `os.rename` when same-device — confirm same-device first).
3. After moving: check destination exists, source gone, size matches.
4. If any check fails: STOP immediately, do not continue the batch, flag this file as a **failed/incomplete move**, surface it to the user explicitly.
5. Only a file that passes gets recorded as a verified, successful move.

## Operations

**SCAN**: Quick audit (summary only) | Full scan (detailed tree) | Compare scans | Detect EXIF/GPS metadata
**ORGANIZE**: Batch rename | Batch MOVE (by date/type/criteria) | Auto-organize | Group by GPS location
**DEDUPE (secondary/cleanup tool)**: Find duplicates that already exist in the library from before this fix, and MOVE them to a review folder (never delete).
**REPORT**: List empty folders (never deleted — reported only)
**ANALYZE**: Size breakdown | Growth patterns | Privacy risks (GPS tagged files)

## Quick Scan (default)

Returns 1-line summary per folder max, as a list:
```
/Volumes/1TB FAM 24: 890GB | 11 folders | 5 empty
Travel: 103GB | 31 folders | 15 empty
```
No verbose logs in chat. Empty folder counts are informational only — never auto-deleted.

## Filename Collision Check (BEFORE every move)

1. Check if a file with the same name already exists at the destination.
2. If no collision: move normally, then verify (Tier 1 process above).
3. If collision: compare EXIF metadata + checksum.
   - Match → same photo. MOVE the incoming file into `_Duplicates_Review/` instead.
   - Differ → rename incoming file with a date/camera disambiguator before moving, flag in preview.
4. Never overwrite. Never delete.

## DEDUPE — cleanup of pre-existing duplicates (secondary tool, MOVE only, never delete)

```
User: "Find duplicates in Travel"
SCAN: Extract EXIF + file checksum for every image
GROUP: Cluster files where checksum matches, OR (EXIF date+camera+GPS+dimensions all match)
PREVIEW (listed): "Found 14 duplicate groups (37 files, 1.2GB)" — one line per group
ASK: "MOVE the 37 redundant copies to _Duplicates_Review/? (yes/no) — nothing is deleted"
Only if explicit YES: mv redundant copies into _Duplicates_Review/, log every decision.
```
Never use filename-only matching to decide duplicates — always confirm with EXIF and/or checksum.

## Smart Grouping Rule — fixed 2-year blocks (default)

- ✅ Default: group years in fixed pairs (`2003-2004`, `2005-2006`...) regardless of file count per year.
- ↪️ **Stretch instead of stranding**: extend the tail block to absorb a small leftover year rather than creating a near-empty standalone folder.
- 📊 Prefer bigger, fuller folders when stretching — "if it can be more, better."
- 🚫 **Naming**: `<Prefix>_<YearStart>-<YearEnd>`, never the word "Grouped".
- ➕ **Re-runs**: merge new files into existing matching block folders, don't create overlapping duplicates.
- Undated files: never silently bucketed into a year block. Ask once per session: leave in place or move to `<Prefix>_Sin_fecha/`.
- Always preview exact year ranges, counts, and average files/folder before asking to proceed.

## Cross-Disk / Multi-Drive Migration (staging-only, see hard constraint above)

1. **Never target a different mounted drive directly as a move destination.** Always stage inside the source drive first: `<SOURCE_DRIVE>/MOVER_A_<TARGET_DRIVE_NAME>/...` (same-device, Tier 1).
2. Bulk-move whole folders when already homogeneously dated. Split mixed folders file-by-file when they span both sides of a date cutoff.
3. Batch large/heavy folders (RAW, video) in chunks (e.g. 300-600 files per pass); make scripts resumable (skip files already absent from source) since a call may hit an execution time limit.
4. Log incrementally (see Log section) — these jobs are the most likely to be interrupted mid-run.
5. Tell the user explicitly, once, that they must relocate each `MOVER_A_<TARGET_DRIVE_NAME>` folder to the real target drive themselves — this skill builds the staging folder but cannot complete the physical cross-drive transfer.
6. If resuming after the user says they relocated a staging folder, verify by checking file counts on the real target drive before continuing — don't assume local paths are unchanged.
7. If a duplicate is discovered from a past cross-device attempt (before this rule existed), report it (both paths, confirm sizes match) and let the user decide — do not attempt to delete either copy.

## Batch Operations (Preview Only Until Authorized)

**1. Rename files** — preview first 10 + total count, ask, then `mv` + Tier 1 verification.

**2. MOVE files by criteria** — scan matches, collision check, preview counts/collisions, ask, then move + verify every file, report failures immediately, report (never delete) empty folders left behind.

**3. Organize by pattern with fixed 2-year blocks** — parse years, apply grouping rule (merge into existing blocks on re-run), collision check, preview groupings + averages + stretch rationale, ask, then move + verify.

**4. Organize by date (EXIF) with fixed 2-year blocks** — same as above, driven by EXIF year.

**5. REPORT: Empty folders** — find, list, never delete.

## EXIF & GPS Operations (Preview Only - MOVE, not copy, not delete)

Scan for GPS metadata, organize by GPS location (same fixed-block proximity logic), export EXIF to CSV — all preview-first, all MOVE-only (Tier 1), all verified. No delete option anywhere in this skill.

## Log — single file at the drive root, written incrementally

```
<DRIVE_ROOT>/DISK_OPERATIONS_<YYYYMMDD>.log
```

**Write each entry to disk as soon as that file's move is verified — never buffer log lines until the end of a batch/script.** Append (open, write, flush) per file or small sub-batch. A killed process must still leave an accurate trail matching real disk state.

Each entry: timestamp, operation type, source path, destination path, verification tier (TIER1-SIZE; TIER2-CHECKSUM is now effectively unused for moves since cross-device moves aren't attempted), result (PASS/FAIL/SKIP), and grouping rationale where relevant.

If a gap is found later (disk state has more moved files than the log recorded), write a `RECONCILIACION` entry explaining it. If a cross-device duplicate incident is found (see hard constraint), write an `INCIDENTE` entry with both file paths and sizes.

## Safety Protocol (ALL operations)

1. **SCAN** → identify what would change
2. **COLLISION CHECK** → EXIF/checksum against destination
3. **PREVIEW** → listed, concise: counts + paths + collisions + grouping rationale
4. **EXPLAIN** → MOVE not COPY; nothing deleted; if cross-disk requested, explain staging-only limitation up front
5. **ASK** → "Proceed? (yes/no)"
6. **WAIT** → explicit authorization required
7. **EXECUTE** → MOVE only if authorized, same-device only
8. **VERIFY** → Tier 1 check; stop and flag on failure
9. **REPORT** → empty folders listed, never removed
10. **LOG** → write each verified action incrementally

**NEVER auto-proceed. ALWAYS require explicit yes/no.**
**ALWAYS use MOVE (mv), NEVER use COPY (cp).**
**NEVER delete anything — no files, no duplicates, no empty folders. Ever.**
**NEVER attempt a move where source and destination are on different mounted drives — build a same-device staging folder instead and tell the user to relocate it themselves.**
**NEVER decide two files are duplicates from filename alone — always confirm with EXIF and/or checksum.**
**Log incrementally, not at the end of a batch.**
**Default grouping is fixed 2-year blocks, never "Grouped" in a folder name, never strand a small trailing year alone.**
**All chat output must be concise, precise, and listed.**

## Examples

**User**: "Move everything before 2022 from WD2TB to my 1TB FAM 24 drive, everything 2022+ to Kingston"
→ Explain up front: staging folders only, user must relocate physically
→ Build `WD2TB/MOVER_A_1TB_FAM_24/` and `WD2TB/MOVER_A_KINGSTON_2TB_26/`, route files by date (Tier 1, same-device, fast)
→ Log incrementally throughout
→ Tell user: "Ready — drag these two folders to their real drives when convenient"
→ If user later relocates them mid-job themselves, verify by checking the real target drive's file counts before resuming/reconciling

**User**: "Organize my Videos by year (2024_*, 2023_*, 2021, 2020, 2018)"
→ Fixed 2-year blocks: 2024+2023 → "Videos_2023-2024" (245); 2021+2020 → "Videos_2020-2021" (13); 2018 flagged standalone if no good pairing.
→ Move + Tier 1 verify each, log incrementally, ask before executing.

**User**: "Find duplicates in Travel"
→ SCAN: EXIF + checksum → PREVIEW: 14 groups, 37 files → ASK → MOVE to `_Duplicates_Review/` if YES, nothing deleted.

**User**: "Quick scan my Downloads"
→ "Downloads: 2.3GB | 156 files | 3 empty folders (reported only, not deleted)"
