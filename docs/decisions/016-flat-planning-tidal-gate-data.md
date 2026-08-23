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
values.

The row contains no Location name, latitude, longitude, geometry, source
provenance, accuracy assessment, review state, readiness state, revision,
history or tombstone. File-level contract, version and update metadata may sit
outside the rows.

Location Editor remains the only spatial editor. Planning's Tidal Gate Data tab
joins row IDs to `tidalGate` Locations for display and derives a read-only Google
Maps link from Location geometry. Adding a row selects an existing gate
Location. Updating or deleting a row changes Planning constants only; it never
creates, edits, reclassifies or deletes a Location.

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

Removing the four directional slack fields is a breaking flat-data change, so
the durable catalogue and export/transfer contracts advance to version 3. A
v0.8 version 2 catalogue or transfer can migrate only when its flood and ebb
spring slack values are equal and its flood and ebb neap slack values are
equal. If either pair is asymmetric, Planning rejects the catalogue or import
before persistence. It must not average the pair or choose one direction as
authoritative without an explicit operator correction. Startup leaves the
original catalogue unchanged; replacement import and merge leave the current
Planning catalogue unchanged.

Planning's OpenAPI contract explicitly documents both the version 3 transfer
and the accepted v0.8 version 2 compatibility-import envelope. Compatibility
is a bounded migration contract, not implicit row-shape detection.

The in-process Planning service advances to
`ajrm-marine-planning-service-v2`, contract version 2. Its gate-list,
gate-lookup and catalogue methods expose the version 3 shared-slack row shape;
consumers must select this explicit service contract rather than interpreting
which row version they received.

Existing Planning catalogues are converted once. Values that map exactly to the
flat calculation model are retained; `source` text becomes `notes`. Evidence,
review, readiness, revision, history and tombstone fields are dropped. A legacy
record missing any required calculation value is omitted and reported rather
than completed by inference. Bundled defaults initialise only an absent file,
so a row deleted by the operator is not recreated on restart.

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
