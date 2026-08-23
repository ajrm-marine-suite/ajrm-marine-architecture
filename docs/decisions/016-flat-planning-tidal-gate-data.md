# ADR-016: Keep tidal-gate planning data flat

Status: accepted  
Date: 2026-08-23

## Context

ADR-015 moved tidal-gate data out of Tidal Database, but its first Planning
model retained evidence, review, readiness, revision, history and tombstone
structures. Those structures made a set of editable calculation constants much
more complicated than the passage calculation or its operator workflow needed.

The spatial and calculation responsibilities are already separate. Location
Editor owns a gate's stable ID, name, type and geometry. Planning needs only the
values used by the calculation and the exact Location ID of its reference port.

## Decision

Marine Planning stores one flat constants row keyed by each tidal-gate Location
UUID. A row contains exactly:

- `referencePortLocationId`;
- `floodSet` and `ebbSet`;
- `springPeakFlow` and `neapPeakFlow`;
- flood and ebb spring/neap offsets after high water;
- shared `springSlack` and `neapSlack` total centred durations, each used for
  both the flood and ebb calculation; and
- free-text `notes`.

The row therefore has six timing fields: four direction-specific HW offsets
and two shared slack durations. It does not store separate flood and ebb slack
values. All six timing fields use canonical minute-precision `HH:MM` with at
least two hour digits. The four offsets may be signed; the two slack durations
are non-negative. Seconds are not part of the current row contract.

The row contains no Location name, latitude, longitude, geometry, source
provenance, accuracy assessment, review state, readiness state, revision,
history or tombstone. File-level contract, version and update metadata may sit
outside the rows. Current `notes` remains editable, but every bundled or
migrated row starts with it blank; legacy source or Notes text is not retained.

Location Editor remains the only spatial editor. Planning's Tidal Gate Data tab
joins row IDs to `tidalGate` Locations for display and derives a read-only Google
Maps link from Location geometry. Adding a row selects an existing gate
Location. Updating or deleting a row changes Planning constants only; it never
creates, edits, reclassifies or deletes a Location.

The Tidal Gate Data table renders this wide row compactly. Longer calculation
labels use two-line headings; this is presentation only and does not add fields
to the durable row.

The reference port is selected from Locations classified
`tidalStandardPort`, and Planning stores that exact Location UUID. Tidal
Database is not the dropdown authority and stores no gate row. At calculation
time Planning joins the selected UUID to Tidal Database's prediction-port data,
reference levels and predictions. A missing runtime join makes the calculation
unavailable; it does not change or expand the stored row.

Planning exports, replaces and merges only its UUID-keyed flat rows. Location
catalogues are transferred separately through Location Editor. Replace removes
local rows omitted from the incoming file. Merge inserts incoming-only rows,
lets incoming rows update matching IDs and retains local-only rows.

Removing the four directional slack fields was a breaking flat-data change and
advanced the durable catalogue and export/transfer contracts to version 3 in
v0.9. Version 4 keeps the exact shared-slack row fields but makes all six timing
values canonical `HH:MM`. The current durable catalogue, export and bundled
seed contracts are therefore version 4.

A v0.9/version 3 catalogue or transfer can migrate only when all four old HW
offsets and both shared slack values end in `:00`. A v0.8/version 2 catalogue or
transfer has the same whole-minute requirement for every old timing value and
can migrate only when its flood and ebb spring slack values are equal and its
flood and ebb neap slack values are equal. Conversion removes the zero seconds
component and pads the hour to at least two digits. It never rounds, and it
clears legacy Notes. If a seconds component is non-zero, or either v0.8 slack
pair is asymmetric, Planning rejects the catalogue or import before
persistence. Startup leaves the original catalogue unchanged; replacement
import and merge leave the current Planning catalogue unchanged.

Planning's OpenAPI contract explicitly documents the current version 4
transfer and the accepted v0.9/version 3 and v0.8/version 2 compatibility
envelopes. Compatibility is a bounded migration contract, not implicit
row-shape detection.

The in-process Planning service advances to
`ajrm-marine-planning-service-v3`, contract version 3. Its gate-list,
gate-lookup and catalogue methods expose canonical minute-precision values
through `ajrm-marine-planning-gate-rows-v3`; consumers select these explicit
contracts rather than interpreting which row version they received.

Existing v0.7 Planning catalogues are converted once. Only values that map
exactly to whole minutes in the flat calculation model are retained; Planning
does not round. Legacy `source`, legacy `notes` and structured source-review
text are discarded, so migrated Notes start blank. Evidence, review, readiness,
revision, history and tombstone fields are dropped. A legacy record missing any
required calculation value is omitted and reported rather than completed by
inference. Bundled defaults also have blank Notes and initialise only an absent
file, so a row deleted by the operator is not recreated on restart.

Omission is reserved for genuinely incomplete v0.7 records. A complete v0.7
record with different flood and ebb spring slack or different flood and ebb
neap slack is a blocking conflict, not an incomplete record. It stops the whole
startup conversion and preserves the original catalogue unchanged. The same
rule applies to a complete v0.8 record.

Planning continues to serialize row writes and transfers. Location Editor may
use Planning's mutation guard to reject a Location candidate that would orphan
a live row. The ordinary workflow is therefore explicit: delete the Planning
row first, then change or delete its Location in Location Editor if desired.

The bounded handoff from an older Tidal Database remains safe without adding
row history. A delayed live row may replace an unchanged bundled default or
fill an absent row, and a delayed tombstone may remove an unchanged bundled
default. A row that differs from the bundled default is treated as local and
preserved. The complete candidate catalogue is persisted and verified before
the old owner is acknowledged. A complete incoming row with asymmetric
directional slack blocks the whole handoff before persistence, leaves the
current Planning catalogue unchanged and is not acknowledged, preserving the
retired source for explicit correction.

## Consequences

- The visible editor and durable row match the calculation inputs directly.
- Spring and neap each have one slack duration, applied consistently to both
  flood and ebb turns.
- Every current timing value is a compact canonical whole-minute value; legacy
  seconds are accepted only when zero and are never rounded.
- Notes remain available for current operator edits without importing old
  source text into bundled or migrated rows.
- Gate positions and reference-port choices remain Location-owned, while tide
  predictions and reference levels remain Tidal Database-owned.
- A constants delete cannot accidentally delete spatial data.
- Exports are small and contain no duplicated coordinates or hidden review
  model.
- Revision histories and tombstone-based restore/conflict semantics from the
  first ADR-015 implementation are deliberately retired.
- Conversion is intentionally fail-closed: incomplete legacy records remain
  absent until an operator supplies all required fields.

This decision refines ADR-015 and supersedes its definition revision/history,
joined whole-gate CRUD, coordinated spatial deletion and tombstone transfer
rules. ADR-015's ownership split remains in force.
