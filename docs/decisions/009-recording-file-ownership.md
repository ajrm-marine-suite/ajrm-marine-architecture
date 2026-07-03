# ADR-009: Align recording files, voyage manifests, and file-selection UI

Status: proposed  
Date: 2026-06-22

## Decision

Use one suite-wide recording file model across Signal K Logger, AJRM Marine
Capture, and AJRM Marine Voyage Viewer.

Signal K Logger owns raw recording files:

- `captures/`: full `.jsonl` / `.jsonl.gz` capture segments;
- `clips/`: extracted `.jsonl` / `.jsonl.gz` clips;
- compression of completed raw recordings;
- playback, delete, download, and clip extraction.

AJRM Marine Capture owns voyage orchestration and voyage metadata:

- movement-triggered start/stop;
- voyage comments and start/stop reasons;
- snapshots, indexes, system context, and incident/debug context;
- optional portable voyage bundles for sharing off the Signal K server.

AJRM Marine Voyage Viewer is a read-only consumer:

- lists voyages, clips, and logs with the same simple selection model;
- plots and exports GPX from any selected voyage, clip, or log;
- does not own raw recording lifecycle, compression, playback, or deletion.
- may write disposable `.ajrm-marine-plot.json` and `.gpx` sidecar caches
  beside source recordings for fast redraw/export, but those sidecars are never
  authoritative.

All three webapps should use the same interaction pattern for files:

1. tabs for `Voyages`, `Clips`, and `Logs` where relevant;
2. a simple selectable file list;
3. action buttons outside the list that operate on the selected file;
4. tactile 3D button styling and visible progress/working states.

## Rationale

The suite currently duplicates file-list behaviours across apps. Per-row action
buttons, slightly different tabs, and inconsistent compression assumptions make
the system harder to explain and harder to use on small boat displays.

The raw Signal K data should have one owner. Signal K Logger already writes,
compresses, replays, and extracts `.jsonl` / `.jsonl.gz` files, so other apps
should treat those files as durable recording sources rather than inventing
parallel storage rules.

Voyages add useful context but should not always need to duplicate large raw log
files. A voyage can often be represented by a manifest/index that references
the appropriate Signal K Logger segments plus snapshots and metadata.

## Consequences

Voyage Viewer must analyse all supported recording sources:

- zipped portable voyage bundles;
- Signal K Logger clips;
- Signal K Logger logs.

AJRM Marine Capture should support two voyage output modes:

- **Manifest mode**: compact voyage metadata references existing logger files by
  stable relative path, time range, size, checksum, and optional segment index.
- **Portable bundle mode**: embed copies of the required raw recording segments
  plus snapshots and indexes for transfer to another machine.

Signal K Logger remains responsible for gzip compression. AJRM Marine Capture
and Voyage Viewer should read `.jsonl` and `.jsonl.gz`, but should not create a
new compression convention.

## Follow-up

- Refactor Signal K Logger and AJRM Marine Capture file browsers to the shared
  “select row, then act” pattern.
- Add a shared CSS/interaction snippet or copied house-style block for file
  tabs, selected rows, and 3D action buttons.
- Add manifest-mode voyage output after portable zip compatibility remains
  proven.
- Keep portable voyage bundles as an explicit export/sharing option.
