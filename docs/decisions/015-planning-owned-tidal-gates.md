# ADR-015: Marine Planning owns tidal-gate definitions

Status: accepted  
Date: 2026-08-23

## Context

A tidal gate has two distinct records with different ownership and lifecycles:

- a spatial Location, which identifies and positions the named gate; and
- planning constants, which describe the gate's reference event, turns, stream
  directions and rates, slack semantics, readiness, provenance and review.

Storing both records in Tidal Database made that plugin an unnecessary owner of
passage-planning definitions. It also coupled editing and transfer of fixed
planning constants to provider credentials, prediction caches and tidal-port
resolution.

## Decision

Location Editor remains the sole durable spatial owner. For a tidal gate it owns
the stable Location ID, name, `tidalGate` classification, coordinates or other
geometry, spatial provenance and Location revision history. Gate locations use
the normal Location CRUD and Location catalogue export, import and merge
contracts.

Marine Planning is the sole durable owner of tidal-gate definitions. Each
definition is keyed by one exact Location ID and contains the explicit,
versioned constants needed for gate-passage planning. Marine Planning owns:

- gate-definition validation, revision history and review state;
- add, update and delete operations;
- JSON export, replacement import and merge by stable Location ID; and
- gate-passage calculations that combine those constants with explicit tidal
  predictions, weather and user inputs.

Tidal Database owns tide-provider configuration, stations, standard and
secondary prediction-port definitions, secondary-port corrections, persistent
caches, tidal-region relationships and tidal prediction calculations. It owns
no tidal-gate definition, operational gate allow-list, gate revision, gate CRUD
or gate transfer format.

Marine Planning presents the joined tidal-gate management workflow. It writes
the spatial part through Location Editor and writes the constants through its
own gate-definition store, then reads each owner back before reporting success.
It does not bypass either owner by editing another plugin's data file.
All Planning gate mutations enter one server-side coordinator before any async
join validation. Whole-gate deletion holds that barrier while Planning writes
its tombstone and Location Editor removes the `tidalGate` type, preventing an
already-validating import or restore from persisting after the spatial record
has gone.
When both services are running, Location Editor's public CRUD, restore,
replacement import and merge operations enter the same coordinator and must
retain the `tidalGate` join for every live Planning definition. The trusted
delete path checks both spatial revision and edit identity atomically.

Cross-store joins use the exact Location UUID. A name, source label or coordinate
must never be treated as the identity of a gate. A gate definition whose
Location is missing or is not classified `tidalGate` is explicit degraded data
and is unavailable for calculation. A reference prediction port that cannot be
resolved through Tidal Database is likewise unavailable rather than guessed.

Creating a gate writes and verifies its Location before its Planning definition.
Deleting a gate removes and verifies the Planning definition before changing
the Location, without releasing the Planning mutation barrier between those
writes. Location Editor's shared spatial service removes the `tidalGate`
classification and deletes the spatial record only when it has no other
Location type to preserve. This order means a partial failure leaves a visible
orphan Location rather than an apparently usable definition with no spatial
identity; after success, later writes fail the exact Location/type join.
Deleting a joined Location with no saved constants still records a first
Planning tombstone, so a seed, transfer or delayed legacy handoff cannot
recreate constants for the removed spatial identity.

Planning gate transfers contain constants, revisions, history and tombstones
only; they contain no Location names, coordinates, geometry or snapshots.
Spatial records are transferred independently through Location Editor before
the Planning constants are imported. A gate-only Location subset must never be
sent to Location Editor's whole-catalogue replacement import, because that
would remove unrelated Locations.

Existing Tidal Database gate definitions may be read only by a bounded,
explicit migration into Marine Planning. The migration must preserve stable
Location IDs and surface conflicts. There is no continuing fallback, dual write
or inferred name-based compatibility mode after migration. Any non-seed
Planning event wins over a delayed legacy event for the same Location ID; that
preserved-local decision is recorded before the old owner is acknowledged.

## Consequences

- Gate constants remain editable and transferable without provider access or a
  populated tide cache.
- Location edits and gate-definition edits retain separate owner revisions and
  can be verified independently.
- Coordinated deletion is linearized across Planning mutations while Location
  Editor remains the sole implementation and durable owner of the spatial write.
- Tidal Database has one coherent tide-prediction responsibility and no
  passage-planning catalogue.
- Marine Planning diagnostics and BITE must check definition-to-Location and
  definition-to-reference-port joins, including missing, duplicate and
  conflicting records.
- Exported gate definitions remain portable because stable Location IDs, units
  and semantics are explicit; missing companion Locations are reported rather
  than reconstructed from names or coordinates.
