# Smart Maya Tidy

A free, open-source [Claude Skill](https://docs.claude.com/en/docs/build-with-claude/skills) that
audits a Google Drive account for duplicate files and storage bloat, explains exactly what it
found and why, and only removes anything after you explicitly approve it — moving files to
Trash, never permanently deleting.

Built by [Smart Maya](https://smartmaya.ai).

## The problem this solves

"My Google Drive is full and I have no idea what's taking up the space" is one of the most
common storage complaints there is. In practice it's almost always one of a few repeatable
patterns: a folder got copied wholesale into a sibling folder at some point, an old computer or
account backup got dumped into Drive and overlaps with what's already there, or the same
photo/video library got exported or extracted more than once. Finding that by hand across tens
of thousands of files isn't realistic — but it's exactly the kind of enumeration-and-pattern-match
task an agent with Drive access can do in one sitting.

## How it works — and an honest note on "enterprise-grade"

This uses the same core idea real deduplication and backup systems use: fingerprint content
instead of comparing every file to every other file, and bucket by that fingerprint so the whole
thing runs in roughly linear time instead of quadratic. Concretely:

1. **Enumerate everything.** Every file in the Drive, not just one folder, via the Google Drive
   connector.
2. **Fingerprint by exact byte size.** A byte-for-byte size match between two files sitting in
   *unrelated* folders is a strong signal — for any file above a trivial size, an accidental
   collision is statistically implausible. This is the same size-prefilter real dedup engines run
   before the expensive step of full content hashing.
3. **Split by confidence.** Matches ≥100KB (or smaller files whose names are obvious copies of
   each other) are treated as high-confidence. Anything smaller with unrelated filenames is set
   aside for manual review, because at a few bytes to a few KB, unrelated files can coincidentally
   land on the same byte count.
4. **Report first.** An XLSX audit — duplicate clusters ranked by space wasted, the largest
   individual files (often a faster win than dedup — big and untouched in years), and a
   folder-by-folder breakdown of where the space actually is.
5. **Approve, then remove — in small batches, to Trash.** Nothing is deleted until you say so,
   itemized. Removal moves files to Google Drive's Trash (recoverable for about 30 days), and
   happens in small, visible batches rather than one unattended mass sweep.

**What this is not:** genuine enterprise dedup (Dell/EMC Data Domain, Commvault, Pure Storage,
etc.) does content-defined block-level chunking and cryptographic hashing, often inline at write
time, on specialized storage hardware, at petabyte scale. This skill borrows the *idea* — fingerprint
and bucket, don't brute-force compare — but the fingerprint here is exact file size (with real
content hashing only where a file is small enough to download and check cheaply), not a
cryptographic hash of every byte of every file. That's a legitimate, honest technique — it's what
lets this run in one sitting against a real account instead of requiring specialized
infrastructure — but it's not the same league as the products above, and this project doesn't
claim otherwise.

## Install

Drop this repository's contents into your Claude Skills directory (or upload it wherever your
Claude client — Claude Code, Cowork, etc. — loads project/personal skills from). The skill
activates when you ask about a full Google Drive, duplicate files, or wanting to declutter/free
up space, provided your Claude session has a Google Drive connector (MCP) enabled.

## Use

Just talk to Claude naturally: *"My Google Drive is nearly full, can you find what's duplicated?"*
Claude will enumerate your Drive, run the clustering analysis (`scripts/dedupe_drive.py`), and
send you back an XLSX report. Nothing gets touched until you review it and say what to remove.

You can also run the analysis script standalone against a JSONL export of file metadata:

```bash
python3 scripts/dedupe_drive.py --input files.jsonl --out report.xlsx
```

`files.jsonl` is one JSON object per line with at minimum `id`, `title`, `fileSize`, `path`,
`createdTime`. See `SKILL.md` for the exact field expectations and how an agent should gather
this from a live Drive connector.

## Safety notes

- Read-only until a report has been delivered and you've explicitly approved specific items.
- Removal always goes to Trash — never a permanent delete.
- The low-confidence (small file, unrelated name) tier is flagged for your manual review, never
  auto-actioned.
- No warranty. Review the report before approving anything, especially on a Drive you haven't
  audited before. You are responsible for what you approve.

## License

MIT — see `LICENSE`. Contributions welcome.
