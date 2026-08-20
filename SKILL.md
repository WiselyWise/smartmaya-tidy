---
name: smartmaya-tidy
description: "Audit a connected Google Drive for likely duplicate files and storage bloat. Trigger on: 'my Google Drive is full', 'find duplicate files', 'storage is full', 'declutter my Drive', 'why is my Drive so full', 'free up Google Drive space', or requests for a duplicate-file or storage audit. Produces an XLSX report only. This skill is read-only and must never delete, move, rename, or modify Drive files."
license: MIT
---

# Smart Maya Tidy — Google Drive duplicate and storage audit

Audit a connected Google Drive for likely duplicate files and storage bloat. Produce a clear XLSX report that helps the user decide what to review. This is a **read-only** workflow.

## Non-negotiable safety boundary

Never delete, trash, move, rename, edit, share, or otherwise modify a Google Drive file or folder.

Do not offer to perform cleanup after the report. If the user asks to remove files, explain that this skill only provides an audit and they must take any cleanup action themselves in Google Drive or use a separately approved workflow.

## When to use

Use for requests such as:

- “My Google Drive is full.”
- “Find duplicate files in my Drive.”
- “Audit our shared Drive before a migration.”
- “Where is our Google Drive storage going?”

## Workflow

1. **Confirm the scope.** State that the audit is read-only and will not alter the Drive. Clarify whether to inspect the accessible Drive root, a named folder, or the files visible to the connector.
2. **Gather metadata only.** Use the configured Google Drive connector’s safe listing and search operations. Collect available file metadata such as ID, title, MIME type, byte size, path or parent, created/modified time, and owner where available. Do not call any connector operation that changes Drive data or permissions.
3. **Record coverage.** Note inaccessible folders, unsupported file types, missing file sizes, pagination limits, and any connector limitations.
4. **Identify likely duplicates.**
   - Group files by exact byte size.
   - Treat groups of files at least 100 KB as likely duplicate candidates when their names or locations support the finding.
   - Flag small or unrelated-name size matches as low confidence, never as confirmed duplicates.
   - Exclude Google-native Docs, Sheets, and Slides from size-based clustering when no byte size is available.
5. **Run the local analysis.** When metadata has been saved as JSONL, run:

```bash
python3 scripts/dedupe_drive.py --input files.jsonl --out report.xlsx
```

6. **Deliver an XLSX report.** Include, where data allows:
   - Summary and coverage notes
   - Duplicate clusters, confidence, and estimated reclaimable space
   - Folder rollup
   - Largest individual files
   - Likely wholesale folder duplication
   - Coverage and gaps
   - A prioritised action plan labelled as recommendations only
7. **Explain limitations.** File-size and filename patterns are triage signals, not cryptographic proof of identical contents. Recommend that users verify high-impact findings before they take action themselves.

## Reporting language

Use “likely duplicate,” “candidate,” and “recommendation” unless byte-identical proof is available from a separate, explicitly approved verification process. Never imply that a file was removed or that the audit frees space.

## Output

Return the XLSX report and a concise summary of:

- total files and folders covered;
- potential duplicate clusters and estimated space impact;
- low-confidence findings requiring manual review; and
- coverage gaps or limitations.

## Privacy

Do not expose file names, paths, ownership details, or report contents beyond the user’s current conversation and the approved local report artifact.
