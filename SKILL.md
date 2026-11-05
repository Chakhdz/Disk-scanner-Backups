---
name: "disk-scanner-carlos"
description: "Organize files by MOVING (not copying) to folders, group into fixed 2-year blocks by default (stretching to absorb small trailing years, never leaving a near-empty standalone year folder), verify moves proportional to real risk, log every operation incrementally to a single DISK_OPERATIONS log at the drive root. Cross-disk migrations build a same-device staging folder only — the sandbox cannot delete files on mounted external drives, so true cross-disk moves must be finished by the user physically. NEVER deletes anything."
---

## Golden Rule

🔴 **NO AUTO-EXECUTION | ALWAYS MOVE, NEVER COPY (except SSD→HDD, see below) | NEVER DELETE ANYTHING | MOVE THE WHOLE FOLDER, NOT JUST ITS CONTENTS**

Every operation shows preview first. Nothing happens without explicit user authorization.
All file operations use **mv** (MOVE), not cp (copy). This skill NEVER deletes files or folders — not originals, not duplicates, not empty folders. Everything that needs to go somewhere gets MOVED, never removed. Empty/leftover folders are moved to `PARA_BORRAR/` at the drive root (keeping their relative path) so the user deletes them by hand.

**Whole-folder rule (user rule):** when moving, staging or consolidating a folder, move the folder itself (with its name and its internal structure), never just the files inside it. Never leave the emptied original folder behind. Only rename files inside it when the user asked for it, and keep the folder as the unit that travels. If the user asked to rename the files, rename them inside the folder and move the folder.

**SSD vs HDD rule (user rule):**
- **SSD → SSD (or SD/flash): MOVE** the whole folder; the original does not stay behind.
- **SSD → HDD: COPY** the whole folder and keep the original on the SSD (the HDD is the backup), unless the user says explicitly that it must be moved. Verify every file with `cmp` after copying and log it as COPY. If a first comparison shows differences, re-copy those files, compare again and write a `RECONCILIACION` line.
- The SSD/HDD type of each disk comes from the dashboard `mapa-discos` (KIND) or from the random-read measurement (about 0.2 to 0.3 ms is SSD, about 10 ms is HDD).
- When the HDD is not connected, prepare the whole folder as a copy in `<SOURCE_DRIVE>/PARA_MOVER_A_<HDD_NAME>/` and copy it when the HDD is connected.

**SISTEMA rule (user rule):** every disk (SSD, SD or HDD) has one folder `SISTEMA/` at its root. All logs and reports about that disk live inside it, never loose at the root:
- `DISK_OPERATIONS_<YYYYMMDD>.log` (the operations log; see Log section)
- scan reports (`DISK_SCAN_*.txt`, `DISK_SCAN_videos_*.txt` and any similar report)
- the disk's own tools and assets (icon, backups of scripts such as `Faceindex.icns`, `*.bak-*`)
When a log or report is found at the root, MOVE it into `SISTEMA/` (same-device rename, log the move). If a file with the same name already exists in `SISTEMA/`, merge by appending (older content first) and move the leftover to `PARA_BORRAR/SISTEMA/`. Exceptions that stay at the root: `Faceindex.command` (the disk button) and `_FACE-INDEX/`.

Flow: Show preview → Ask confirmation → Wait for yes/no → Execute only if authorized → **Verify move (see Verification Tiers)** → **Log incrementally**

Once the user has approved a rule (e.g. the rules in "Naming & grouping rules"), applying it to further folders of the same job still gets a short preview with counts (a dry run), but the user's explicit "sí" for the rule counts as the approval for that preview if nothing in it differs from the rule.

## ⚠️ HARD ENVIRONMENT CONSTRAINT: cannot delete files on mounted external drives

Confirmed by direct test: the sandbox this skill runs in can CREATE and WRITE files on connected/mounted external drives, but CANNOT delete/unlink files on them via any shell tool available (`rm`, `os.unlink`, `shutil.move`'s fallback delete step all fail with `PermissionError: Operation not permitted`). This is a hard sandbox restriction, not a skill policy choice.

**Consequence for cross-device operations:**
- A move WITHIN the same disk/filesystem (same `st_dev`) works fine — it's an atomic rename, no delete step involved. Use this freely (Tier 1).
- A move ACROSS two different disks (e.g. from `DISK_A` to `DISK_B`) requires copy-then-delete-original. The copy will succeed; **the delete-original step will fail every time.** Do NOT attempt this directly — it silently leaves a verified-good copy at the destination but ALSO leaves the original in place, becoming an unintended duplicate that nothing flags automatically unless you check for it.
- **Never call `shutil.move`/`mv` with source and destination on two different mounted drives.** Check `os.stat(src).st_dev != os.stat(dest).st_dev` first — if they differ, this is cross-device and cannot be completed as a real move by this skill.

**What to do instead for multi-drive migrations:**
1. Build a staging folder for each target drive INSIDE the source drive itself (e.g. `<SOURCE_DRIVE>/MOVER_A_<TARGET_DRIVE_NAME>/`), organized exactly as it should land on the target, **with each folder moved whole (name and structure intact)**. This is a same-device move — fast, safe, fully completable by this skill.
2. Tell the user clearly that the skill cannot physically move data between two separate mounted drives, and that they need to drag/copy the staging folder to the real target drive themselves (Finder, Explorer, etc.), then the skill can help verify/clean up afterward if asked.
3. If the user says they already relocated a staging folder to its real drive (this happens mid-job on large transfers), **verify by checking file counts on the real target drive** before continuing or reconciling logs — don't assume the local staging path still exists, and don't try to re-verify by re-hashing across drives unless asked.
4. If a direct cross-device move was attempted before this was known and left a duplicate (copy succeeded, original delete failed), log it as an incident (see Log section), tell the user exactly which file(s) and both locations, and let them decide whether to remove the duplicate themselves — this skill cannot delete it from either drive on its own once it needed a cross-device delete.
5. Never touch existing `MOVER_A_*` / `PARA_MOVER_A_*` staging folders when organizing other folders.

## Communication style — concise, precise, listed

Every preview, clarification, question, or result summary shown to the user must be:
- **Concise**: no filler sentences, no restating context the user already knows.
- **Precise**: exact counts, exact folder names, exact numbers — never vague language like "several" or "many."
- **Listed**: use a bullet or numbered list for anything with more than one item (folders, counts, questions, options). Never bury multiple facts in a paragraph.

Apply this to every step of the Safety Protocol below (PREVIEW, EXPLAIN, ASK, REPORT) — not just final summaries.

## Priority #1: Anti-Duplication Verification — proportional to real risk

**Verification Tiers (pick per move, based on `os.stat(src).st_dev` vs `os.stat(dest_parent).st_dev`):**

- **Tier 1 — Same device (same disk/filesystem):** the only tier this skill can safely execute end-to-end as a true move. After the move: confirm the destination file exists, confirm the source path no longer exists, confirm destination size matches the size recorded before the move. Do NOT hash full file contents — wasted I/O for an atomic rename.
- **Tier 2 — Cross device (different physical disk/mount):** per the hard constraint above, this skill CANNOT complete a true move here (the delete-original step will fail). Do not attempt it as a move. If the user explicitly wants a copy left on both drives (always the case for SSD→HDD), that's a copy operation: copy the whole folder, verify file by file with `cmp`, log it, and tell the user which original they must delete by hand if they ever want it gone.

**Mandatory sequence for Tier 1 (the only tier actually executed as MOVE):**
1. Before moving: record source size.
2. Execute the move (`mv` / `shutil.move`, which uses `os.rename` when same-device — confirm same-device first).
3. After moving: check destination exists, source gone, size matches.
4. If any check fails: STOP immediately, do not continue the batch, flag this file as a **failed/incomplete move**, surface it to the user explicitly.
5. Only a file that passes gets recorded as a verified, successful move.
6. Move the macOS `._<name>` AppleDouble companion together with its file when it exists; ignore `._*` and `.DS_Store` when counting files or deciding a folder is empty. (On this setup renaming a file also renames its `._` companion automatically; the shell cannot read or copy `._` files, and that is fine.)

## Operations

**SCAN**: Quick audit (summary only) | Full scan (detailed tree) | Compare scans | Detect EXIF/GPS metadata
**ORGANIZE**: Batch rename | Batch MOVE (by date/type/criteria) | Auto-organize | Group by GPS location
**DEDUPE (secondary/cleanup tool)**: Find duplicates that already exist in the library from before this fix, and MOVE them to a review folder (never delete).
**REPORT**: List empty folders (reported; moved to PARA_BORRAR only when the user asks)
**ANALYZE**: Size breakdown | Growth patterns | Privacy risks (GPS tagged files)

## Quick Scan (default)

Returns 1-line summary per folder max, as a list:
```
/Volumes/DISK_B: 890GB | 11 folders | 5 empty
Travel: 103GB | 31 folders | 15 empty
```
No verbose logs in chat. Empty folder counts are informational only — never auto-deleted. Any scan report written to the disk goes in `SISTEMA/` (SISTEMA rule).

## Where the date comes from

- ALWAYS use real metadata, never the file modification date: EXIF `DateTimeOriginal` (fallback `Image DateTime`, `DateTimeDigitized`) for photos; `ffprobe` `creation_time` for videos. File mtime is unreliable (e.g. 18,000 old photos can show the copy year).
- Files without metadata date → `FOTOS_Sin_fecha/` (photos) or `Videos_Sin_fecha/` (videos), keeping their original sub-path. Never silently bucket undated files into a year.
- Read metadata with `exifread` (pip install --break-system-packages exifread) or `ffprobe`; skip `._*` and `.DS_Store`. Check path unicode with NFC normalization when comparing folder names (macOS stores NFD).

## Naming & grouping rules (approved by the user)

1. **Year blocks**: keep the blocks the library already uses (e.g. 3-year: `FOTOS_2010-2012`, `FOTOS_2013-2015`...). If none exist, default to fixed 2-year blocks. Names: `<Prefix>_<YearStart>-<YearEnd>`, never the word "Grouped".
2. **250 rule (year folders)**: inside any thematic folder, a year (or merged range) with fewer than 250 files is merged with the NEXT year and the range widened (e.g. `2018-2019` + `2021` → `2017-2021`). Accumulate years in ascending order until the block reaches 250; a small tail merges backwards. This replaces the older 20-file threshold for year folders.
3. **Small groups**: a whole folder/group with fewer than 20 files goes into `Varios (<block>)/<Group> (<years>)/`.
4. **Thematic / mother folders**: keep the mother folder name and append the real year range of its files: `Name (YYYY-YYYY)`, or `Name (YYYY)` for one year. Inside, create one folder per year (or merged range per rule 2). Sub-folders keep their name plus their own range in that year: `Rectoria (2016)`, `Bautizo Catalina (2021)`; when year folders are merged, recompute the sub-folder range (`Canon_0000 (2013-2015)`). Loose files go straight into their year folder.
5. **Respect device/camera names**: if the name carries Canon, GoPro, iPhone, Sony, etc., keep it and add the dates: `2016 Gopro SLP (2016)`.
6. **Remove resolution tokens** (`5184x3456`, `4104x2736`...) from folder AND file names. If two folders end up with the same name, merge their contents without overwriting. If two files end up with the same name: identical (size + cmp) → `_Duplicates_Review/`; different → append the EXIF/creation date to the file name (`_YYYYMMDD-HHMMSS`).
7. **Travel / same-theme folders spread over several places**: find every folder with that name (e.g. `Travel`), merge them into ONE folder `Travel (YYYY-YYYY)` without keeping the original sub-divisions (raw/jpg/year sub-folders). The child folders are named `Travel (years)` (`Travel (2013)`, `Travel (2017-2021)`, `Travel (sin fecha)`) and follow the 250 rule. Undated files go to `Travel (sin fecha)`.
8. **Keep original sub-paths** when merging a source folder into a block (e.g. `FOTOS_2013-2015/SANDISK/<original path>`) unless a rule above says to flatten (Travel).
9. **Leftover folders**: after moving, folders with no real files (only `._*`/`.DS_Store`) are moved whole to `PARA_BORRAR/<same relative path>` after the user says yes; never deleted.
10. **Re-runs**: merge new files into existing matching folders, don't create overlapping duplicates. Always preview exact ranges, counts and per-folder averages before asking to proceed.
11. **Whole folder moves**: when the user says to move, send or consolidate a folder somewhere (another folder on the same disk, a staging folder, a target disk), the folder travels with its name and internal structure; its contents are never poured loose into the destination and no emptied shell is left at the origin. Exception: only when the user explicitly asks to merge contents into an existing folder.
12. **Folders without internal structure**: when a folder holds many files whose names carry a project prefix (e.g. `XPORTS-24_print 2024 nyc-315.png`), group them in subfolders by that prefix, keep each file's name and number, and rename the mother folder so its year range matches what it now holds.
13. **SISTEMA folder**: logs, scan reports and the disk's own tools/assets live in `SISTEMA/` at each disk root (SISTEMA rule in the Golden Rule). Never organize, regroup or rename the contents of `SISTEMA/` by year.

## Filename Collision Check (BEFORE every move)

1. Check if a file with the same name already exists at the destination.
2. If no collision: move normally, then verify (Tier 1 process above).
3. If collision: compare size + byte-compare (and EXIF if needed).
   - Same file → MOVE the incoming file into `_Duplicates_Review/` instead.
   - Differ → rename incoming file with a date disambiguator before moving, flag in preview.
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

## Cross-Disk / Multi-Drive Migration (staging-only, see hard constraint above)

1. **Never target a different mounted drive directly as a move destination.** Always stage inside the source drive first: `<SOURCE_DRIVE>/MOVER_A_<TARGET_DRIVE_NAME>/...` (same-device, Tier 1), moving each folder whole. For SSD→HDD the staged folder is a copy and the original stays (SSD vs HDD rule above).
2. Bulk-move whole folders when already homogeneously dated. Split mixed folders file-by-file when they span both sides of a date cutoff.
3. Batch large/heavy folders (RAW, video) in chunks; make scripts resumable (skip files already absent from source).
4. Log incrementally (see Log section) — these jobs are the most likely to be interrupted mid-run.
5. Tell the user explicitly, once, that they must relocate each `MOVER_A_<TARGET_DRIVE_NAME>` folder to the real target drive themselves — this skill builds the staging folder but cannot complete the physical cross-drive transfer.
6. If resuming after the user says they relocated a staging folder, verify by checking file counts on the real target drive before continuing — don't assume local paths are unchanged.
7. If a duplicate is discovered from a past cross-device attempt, report it (both paths, confirm sizes match) and let the user decide — do not attempt to delete either copy.

## Running long jobs (execution tips)

- A single shell call is limited (~3 min) and background processes die when the call ends (use `setsid nohup` only when the work can resume). Write scripts that are **resumable and time-boxed** (e.g. 150 s budget, skip files already moved or already in the metadata CSV) and call them repeatedly, reporting progress after each pass.
- Scan metadata once into a CSV (path, size, date, year), preview from the CSV (dry run), then move from the same CSV.
- The user may be moving things on the disk at the same time: before a big move, re-list the top level; if folders changed that you did not touch, tell the user and ask whether another process is running.
- Report progress every few minutes during long jobs.

## Batch Operations (Preview Only Until Authorized)

**1. Rename files** — preview first 10 + total count, ask, then `mv` + Tier 1 verification.
**2. MOVE files by criteria** — scan matches, collision check, preview counts/collisions, ask, then move + verify every file, report failures immediately, report (never delete) empty folders left behind.
**3. Organize by pattern / by date (EXIF) with year blocks and the naming rules above** — parse dates, apply grouping rules (merge into existing blocks on re-run), collision check, preview groupings + counts + rationale, ask, then move + verify.
**4. REPORT: Empty folders** — find, list; move to `PARA_BORRAR/` only on request, never delete.

## EXIF & GPS Operations (Preview Only - MOVE, not copy, not delete)

Scan for GPS metadata, organize by GPS location (same fixed-block proximity logic), export EXIF to CSV — all preview-first, all MOVE-only (Tier 1), all verified. No delete option anywhere in this skill.

## Face analysis

Face analysis is done by the user's own tool `face-index` (InsightFace) in `GROK BOT WORKSHOP/LABORATORIO/face-index`, which runs on the user's Mac (not in the Linux sandbox). Do not reinstall it; give the user the command (`.venv/bin/python face_index.py scan "<disk>/FOTOS" --nombre <NAME> --coreml`) and recommend scanning AFTER the reorganization so paths do not change. It writes `_FACE-INDEX/` at the drive root; never move or rename that folder. The disk button is `Faceindex.command` (formerly `BuscaCaras.command`) at the drive root; never move it into a subfolder.

## NSFW scan

The user's NSFW scanner lives in `GROK BOT WORKSHOP/LABORATORIO/nsfw-scan/` (skill `escaner-nsfw`): read-only, writes CSV in its own `resultados/` folder, never inside the scanned disk.

## Log — single file per day inside SISTEMA/, written incrementally

```
<DRIVE_ROOT>/SISTEMA/DISK_OPERATIONS_<YYYYMMDD>.log
```

Create `SISTEMA/` first if the disk does not have it. Older logs found at the root are moved into `SISTEMA/` (SISTEMA rule); never leave a log loose at the root.

**Write each entry to disk as soon as that file's move is verified — never buffer log lines until the end of a batch/script.** Append (open, write, flush) per file or small sub-batch. A killed process must still leave an accurate trail matching real disk state.

Each entry: timestamp, operation type, source path, destination path, verification tier (TIER1-SIZE), result (PASS/FAIL/SKIP), and grouping rationale where relevant.

If a write to the log fails transiently (it happens on exFAT right after creating the file), compare disk state with the log afterwards and append the missing entries with a `RECONCILIACION` line; do not treat a failed log write as a failed move without checking the disk. If a gap is found later, write a `RECONCILIACION` entry explaining it. If a cross-device duplicate incident is found, write an `INCIDENTE` entry with both file paths and sizes.

## Safety Protocol (ALL operations)

1. **SCAN** → identify what would change
2. **COLLISION CHECK** → size/byte-compare/EXIF against destination
3. **PREVIEW** → listed, concise: counts + paths + collisions + grouping rationale
4. **EXPLAIN** → MOVE (SSD→SSD) or COPY (SSD→HDD); nothing deleted; if cross-disk requested, explain staging-only limitation up front
5. **ASK** → "Proceed? (yes/no)"
6. **WAIT** → explicit authorization required
7. **EXECUTE** → MOVE only if authorized, same-device only, whole folders
8. **VERIFY** → Tier 1 check; stop and flag on failure
9. **REPORT** → empty folders listed, never removed
10. **LOG** → write each verified action incrementally into `SISTEMA/`

**NEVER auto-proceed. ALWAYS require explicit yes/no.**
**ALWAYS use MOVE (mv) between SSDs; SSD→HDD is a COPY that leaves the original (unless the user says otherwise).**
**ALWAYS move the folder as a whole, never just its contents.**
**ALWAYS keep logs, scan reports and disk tools inside `SISTEMA/` at each disk root.**
**NEVER delete anything — no files, no duplicates, no empty folders. Ever.**
**NEVER attempt a move where source and destination are on different mounted drives — build a same-device staging folder instead and tell the user to relocate it themselves.**
**NEVER decide two files are duplicates from filename alone — always confirm with size/byte-compare, EXIF and/or checksum.**
**Log incrementally, not at the end of a batch.**
**Dates come from metadata, not from file modification time.**
**All chat output must be concise, precise, and listed.**

## Examples

**User**: "Move everything before 2022 from DISK_A to my DISK_B drive, everything 2022+ to DISK_C"
→ Explain up front: staging folders only, user must relocate physically
→ Build `DISK_A/MOVER_A_DISK_B/` and `DISK_A/MOVER_A_DISK_C/`, route files by date (Tier 1, same-device, fast)
→ Log incrementally throughout in `DISK_A/SISTEMA/`

**User**: "Manda PERSONAL al HDD HDD_D"
→ SSD→HDD: copy the whole folder `PERSONAL ` into `PARA_MOVER_A_HDD_D/`, original stays, verify with cmp, log COPY.

**User**: "Manda la carpeta Expo grafica a FAYD" (SSD→SSD)
→ The whole folder moves with its name and structure (renamed files stay inside it), not its photos loose.

**User**: "Meta esos logs y reportes a sistema"
→ Create `SISTEMA/` at each disk root, move `DISK_OPERATIONS_*.log` and `DISK_SCAN_*.txt` into it (same-device rename), log each move, nothing loose left at the root.

**User**: "Organiza Canon"
→ Read EXIF → `Canon (2000-2021)/2000-2013, 2014-2016, 2017, 2018-2019, 2020, 2021` (years < 250 merged with the next), sub-folders as `Bautizo Catalina (2021)`, resolution removed from names.

**User**: "Junta todas las carpetas Travel"
→ Find every `Travel` folder, merge into one `Travel (2011-2021)/`, no sub-divisions; children `Travel (2011-2012)`, `Travel (2013)`... and `Travel (sin fecha)`.

**User**: "Find duplicates in Travel"
→ SCAN: EXIF + checksum → PREVIEW: 14 groups, 37 files → ASK → MOVE to `_Duplicates_Review/` if YES, nothing deleted.

**User**: "Quick scan my Downloads"
→ "Downloads: 2.3GB | 156 files | 3 empty folders (reported only, not deleted)"