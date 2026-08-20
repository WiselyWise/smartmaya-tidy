---
name: smartmaya-tidy
description: "Audit a Google Drive for duplicate files and storage bloat, and safely clean it up with the owner's approval. Trigger on: \"my Google Drive is full\", \"find duplicate files\", \"clean up my drive\", \"storage is full\", \"declutter my drive\", \"why is my Drive so full\", \"free up Google Drive space\", or any request to find/remove duplicate files in a connected Google Drive. Produces an XLSX audit report before touching anything, and only deletes (to Trash, never permanently) after explicit, itemized owner approval."
license: MIT
---

# Smart Maya Tidy — Google Drive duplicate & storage audit

Finds duplicate files across an entire Google Drive account, explains exactly what it found and
why it's confident, and only removes anything after the owner explicitly approves — moving
files to Trash (recoverable ~30 days), never permanent deletion.

## Why this exists

"My Drive is full of duplicates" is one of the most common storage complaints there is, and it's
usually caused by the same handful of patterns: a folder got copied wholesale into a sibling
folder at some point, an old computer or account backup got dumped into Drive and overlaps with
what's already there, or the same photo/video library got exported/extracted more than once.
None of that requires exotic tooling to find — it requires enumerating everything and looking
for content that matches, at a scale a human won't do by hand.

## Method (and what it honestly is/isn't)

This borrows the core idea professional deduplication and backup systems use — fingerprint
content and bucket by the fingerprint instead of comparing every file to every other file — but
it is **not** the same thing as enterprise block-level dedup (Data Domain, Commvault, etc.),
which does content-defined chunking and cryptographic hashing across the wire in specialized
storage hardware. Be upfront about that difference if a user or reader asks how this compares.

What this skill actually does, in order:

1. **Fingerprint by exact byte size, drive-wide.** Enumerate every file (via the Google Drive
   connector's `search_files`, paginating with `parentId` recursion from the Drive root — see
   "Gathering the data" below) and group them by exact `fileSize`. A byte-for-byte size match
   between two files in *unrelated* folders is a strong signal — for anything above a trivial
   size, an accidental collision is statistically implausible. This is the same size-prefilter
   real dedup engines use before doing full content hashing, because hashing everything is the
   expensive step even for them.
2. **Split by confidence.** Files ≥100KB (or smaller files whose names are clear variants of each
   other, e.g. `IMG_0431.MOV` vs `IMG_0431 (1).MOV`) are treated as high-confidence duplicates.
   Files below that size with unrelated names are set aside for manual review — at a few bytes to
   a few KB, unrelated files can coincidentally share a byte count (ten different tiny text
   files can all happen to be exactly 6 bytes), so don't auto-act on those. `scripts/dedupe_drive.py`
   implements this split; do not skip it when adapting the logic.
3. **Report before touching anything.** Build an XLSX (via the `xlsx` skill/openpyxl) with: a
   summary of what was found, the duplicate clusters sorted by reclaimable space, the largest
   individual files (the "cold storage" angle — big + untouched in years is often a faster win
   than dedup), and a folder-level rollup showing where the space actually is. Send this to the
   user. Do not delete anything yet.
4. **Get explicit, itemized approval.** Tell the user what you're proposing to remove and how
   much space it frees, and wait for a clear go-ahead. Never infer approval from an earlier,
   more general statement like "clean up my drive" — the report is what makes the specific
   decision informed.
5. **Remove in small, visible batches — never mass, unattended deletion.** Even though the
   destination is Trash (recoverable), moving thousands of files in one unattended, fully
   automated sweep is a large blast radius for a mistake to hide in, and some environments'
   safety layers will (correctly) push back on many parallel agents doing bulk destructive
   actions at once. Do the actual trashing yourself, directly, in batches of a few dozen to a
   few hundred, and check in with progress between batches rather than firing off many parallel
   subagents to do it all at once unattended.

## Gathering the data

This skill assumes a Google Drive MCP connector is available (tools typically named
`mcp__Google_Drive__search_files`, `get_file_metadata`, `trash_file`). Enumerate with a query
like `owner = 'me' and mimeType != 'application/vnd.google-apps.folder'`, paginating with
`pageToken`/`nextPageToken`. For very large drives, a single top-down crawl can exceed one
session's practical limits — it's fine (and often necessary) to split the crawl across several
parallel research/sub-agent tasks by top-level folder, each collecting
`id, title, fileSize, mimeType, parentId, createdTime, modifiedTime, viewedByMeTime` for every
file, then merge their output into one dataset before clustering. `scripts/dedupe_drive.py`
expects that merged dataset as a JSONL file, one file record per line, with a `path` field
(full folder path including filename).

If a crawl gets interrupted (rate limits, session limits, timeouts), that's normal at scale —
resume it rather than starting over; don't silently drop partial results.

## Running the analysis

```
python3 scripts/dedupe_drive.py --input files.jsonl --out report.xlsx
```

See `scripts/dedupe_drive.py` for the clustering logic and report structure — read it before
adapting this skill to a different connector or storage backend, since the confidence-tiering
in step 2 above is the part most worth preserving exactly.

## Safety rules (do not skip these)

- Read-only until the report is delivered and the user has explicitly approved specific items.
- Deletions go to Trash, never permanent delete — say so explicitly when asking for approval.
- Never bulk-delete via many parallel unattended agents; do it yourself, in small batches, with
  visible progress.
- Low-confidence (small, unrelated-name) matches are for the user's manual review, not automatic
  action.
- If you're not certain a "confirmed" duplicate really is one (e.g. its size is suspiciously
  common, or the filenames suggest genuinely different content), say so rather than including it
  in a bulk-approval batch.
