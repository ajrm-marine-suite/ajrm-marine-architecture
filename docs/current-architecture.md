# AJRM Marine Current Architecture

Status: canonical current-alpha architecture
Updated: 2026-08-23

AJRM Marine is a set of focused Signal K plugins joined by explicit, versioned
contracts. Backend plugins own domain state; browser apps render it and send
deliberate commands. Signal K remains the public integration layer.

## Active packages and owners

| Package | Authority or role |
| --- | --- |
| Console | Suite shell, setup/help, Alerts view, installed-app visibility and BITE orchestration |
| Traffic | AIS target model, CPA/TCPA, encounter meaning, profiles and collision notification content |
| Vessel Database | Durable vessel identity, classification and operator notes |
| Notifications | Generic notification lifecycle, ordering, deduplication and audio requests |
| Audio | One authoritative audio queue/timeline and all playback/stream delivery |
| Display | Live chart, own vessel, traffic, routes, locations, profiles, tide and weather presentation |
| Navigation Integrity | Navigation Reference, GNSS trust, operational/integrity DR and the DR Plotter |
| Instruments | Instrument presentation, derived instrument paths and instrument alerts |
| Voyages (Capture) | Canonical recording, replay, recapture, recovery, bundles and voyage review |
| Location Editor | Versioned spatial catalogue: IDs, names, types, coordinates/geometry, anchoring attributes and profile areas, including tidal-gate Locations |
| Tidal Database | Tide-provider adapters, stations, port prediction definitions, secondary corrections, caches, tidal-region relationships and tide-prediction calculations; no gate definitions |
| Weather Database | Provider adapters, provider-separated durable caches, nearest-Location resolution and nearest-cache fallback |
| Marine Planning | Durable flat tidal-gate calculation rows, constants-only CRUD and export/import/merge; gate-passage and anchor planning over Locations, tide and weather services |
| Snapshot | Point-in-time suite diagnostics, including shared location/tide/weather/planning evidence |
| Simulator | Explicit test source for vessel, AIS, environment, route and GNSS scenarios |
| Pi Controller | Optional host telemetry and privileged Raspberry Pi helper actions |
| Map Core | Build-time shared map controls, chart cycling and tidal-curve presentation; not a runtime plugin |

The desktop Audio Player is built and distributed by Audio rather than being
another Signal K plugin.

## Ownership rules

- Location Editor is the sole durable owner of location IDs, names,
  classifications and geometry. A tidal-gate definition in Marine Planning
  references its gate Location by exact UUID and never duplicates the name or
  coordinates as identity. Tidal Database keys predictions and region
  relationships by Location IDs. Weather Database may use a Location as
  forecast context but does not copy the catalogue.
- Tidal Database presents Location-owned port and region names through exact
  UUID joins. Any retained label used during temporary Location-service loss is
  an explicitly non-authoritative cache fallback, not a separately editable
  name. Tidal durable definitions version 3 stores that fallback as
  `cachedLocationName`; joined projections report whether their displayed name
  came from `location` or `cached`. Its gate-free service, status and diagnostic
  surfaces use contract major version 2. Standard-port and secondary-port
  classifications are mutually exclusive, and a correction definition is
  operational only while its complete parent chain is valid and
  provider-backed.
- Marine Planning is the sole durable owner of flat tidal-gate calculation
  rows, constants-only CRUD and row export/import/merge. Rows contain no name,
  coordinates, provenance, review state, revision history or tombstone. Their
  six timing values are four direction-specific spring/neap HW offsets plus
  shared `springSlack` and `neapSlack` durations used for both flood and ebb.
  All six use canonical minute-precision `HH:MM` with at least two hour digits;
  only the four HW offsets may be signed. Current `notes` remains editable, but
  bundled and migrated rows start blank. The Tidal Gate Data view keeps this
  wide row compact and uses two-line headings for longer calculation labels.
  The durable catalogue, export and seed contracts are version 4. Its explicit
  in-process surface is `ajrm-marine-planning-service-v3`, contract version 3,
  and its joined gate-row surface is `ajrm-marine-planning-gate-rows-v3`. It
  also owns gate-passage calculations. Tidal Database owns no gate rows or gate
  transfer contract.
- Planning serializes row writes, transfers and legacy conversion. Deleting a
  row changes Planning only. Location changes remain separate Location Editor
  operations.
- Legacy conversion may omit and report a genuinely incomplete v0.7 row, but
  it converts timing values only when they are exact whole minutes and never
  carries legacy source text into Notes. A v0.9/version 3 or v0.8/version 2
  compatibility value must end in `:00`; conversion never rounds and clears
  legacy Notes. A complete v0.7 or v0.8 row with asymmetric flood/ebb slack is
  instead a blocking conflict across startup conversion, delayed Tidal handoff
  and v0.8 compatibility import. The original/current catalogue stays
  unchanged, and a blocked delayed source is not acknowledged. Planning's
  OpenAPI contract documents the accepted version 4, v0.9/version 3 and
  v0.8/version 2 transfer envelopes explicitly. Until acknowledgement, Tidal
  Database continues to serialize the quarantined source as a genuinely
  rollback-readable legacy v1 file; acknowledgement is the v3 cutover.
- Location Editor's public and shared-service mutations join that coordinator
  when Planning is available. A candidate must retain each live gate UUID as a
  `tidalGate` and each live `referencePortLocationId` as a
  `tidalStandardPort`. The operator deletes or updates the Planning row first
  when a later spatial deletion or reclassification is intended.
- Tidal Database and Weather Database alone contact their providers and own
  provider caches. Display and Planning do not implement private provider
  access or caches.
- Display waits until it has resolved a fresh or retained last-known own-vessel
  position before centring and revealing its initial chart or resolving
  automatic environment data. It centres first and only then requests tide and
  weather. After a bounded wait with no usable position it may reveal a no-GPS
  chart, but chart centre is never invented as environment context. Tidal
  Database and Weather Database resolve independently; Weather Database returns
  the selected Location or cached coordinate group and its distance from the
  resolved vessel position.
- Own-position freshness is evaluated from the position sample time with a
  30-second threshold; the longer AIS target-retention period never turns
  an aged own-position sample into a fresh fix. Programmatic startup centring
  preserves vessel follow, while an actual operator pan still pauses it.
- Planning accepts fetched tide events only when the resolver explicitly names
  the exact requested reference-port UUID. The tide response carries the
  reference levels associated with those events, so a refresh cannot mix new
  events with an earlier level snapshot.
- Weather Database coalesces simultaneous work for one provider/cache key,
  replaces cache files atomically with unique temporary paths and reports cache
  fallback according to the selected primary forecast. Planning's ordinary gate
  weather/tide reads are cache-aware; its deliberate provider refresh routes
  require Signal K read/write or administrator access.
- Traffic alone decides encounter meaning and collision wording. Notifications
  transports provider meaning; Audio schedules it; displays do not infer it
  from prose.
- Navigation Integrity publishes the retained Navigation Reference Signal K
  contract as an integrated component. The stable path is current API, not
  support for the retired standalone package.
- Voyages records canonical physical input at `input/sensor-input.jsonl`, plus
  optional recorded outputs and evidence. Old voyage layouts must be converted
  outside the runtime before playback.
- Snapshot reads diagnostics already held by plugins and does not trigger a
  tide or weather provider request.

The tidal-gate ownership split and flat row refinement are recorded in
[ADR-015](decisions/015-planning-owned-tidal-gates.md) and
[ADR-016](decisions/016-flat-planning-tidal-gate-data.md). Position-first
environment presentation and nearest weather-cache selection are recorded in
[ADR-017](decisions/017-position-first-display-environment.md). The explicit
v2 Tidal boundary and cross-owner integrity checks are recorded in
[ADR-018](decisions/018-harden-planning-environment-contracts.md).

## Required and optional runtime

Console requires Display, Traffic, Notifications, Audio, Voyages, Navigation
Integrity, Location Editor, Tidal Database and Weather Database. Planning,
Instruments, Vessel Database, Snapshot, Simulator and Pi Controller are
optional, task-specific additions. Derived Data and Charts Provider Simple are
third-party integrations, not AJRM authorities.

## Persistence

Durable state includes plugin configuration, Locations and their revisions,
Marine Planning tidal-gate rows, Tidal Database port and
region definitions, tidal and weather caches, vessel records, routes, voyage
bundles, DR plots, BITE reports and explicit user preferences. Active traffic,
notification and audio state is session-scoped and rebuilt from fresh input.

## Retired package boundaries

Standalone Alerts, Navigation Reference, DR Plotter, Instrument Alerts,
Harbour Editor, Logger and Voyage Viewer are retired. Their current functions
live respectively in Console, Navigation Integrity, Instruments, Location
Editor and Voyages. Active packages do not import their old configuration or
data models.

## BITE boundary

Console BITE checks installed-app visibility, individual service contracts and
cross-app invariants: Location profile-area consumers; Location/Tidal
port-and-region joins; Planning row joins to gate and reference-port Locations
and Tidal Database prediction ports; Planning's three shared services; Snapshot
shared-data evidence; Traffic → Notifications → Audio → Display flow;
Capture bundle round trips; and Navigation Integrity/DR behaviour. Package unit
tests remain responsible for exhaustive internal calculations.

## Safety

The suite assists situational awareness and testing. It is not a certified
navigation, collision-avoidance, anchoring, tide, weather or vessel-control
system, and the skipper remains responsible for decisions.
