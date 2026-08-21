# AJRM Marine Current Architecture

Status: canonical public-beta architecture  
Updated: 2026-08-21

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
| Display | Live chart, own vessel, traffic, routes, locations, profiles and tide presentation |
| Navigation Integrity | Navigation Reference, GNSS trust, operational/integrity DR and the DR Plotter |
| Instruments | Instrument presentation, derived instrument paths and instrument alerts |
| Voyages (Capture) | Canonical recording, replay, recapture, recovery, bundles and voyage review |
| Location Editor | Versioned spatial catalogue: names, types, geometry, anchoring attributes and profile areas |
| Tidal Database | Provider adapters, station cache, port prediction definitions, tidal-region relationships and tide calculations |
| Weather Database | Provider adapters, provider-separated durable caches and forecast resolution |
| Marine Planning | Read-only use of Locations, Tidal Database and Weather Database for gate and anchor planning |
| Snapshot | Point-in-time suite diagnostics, including shared location/tide/weather/planning evidence |
| Simulator | Explicit test source for vessel, AIS, environment, route and GNSS scenarios |
| Pi Controller | Optional host telemetry and privileged Raspberry Pi helper actions |
| Map Core | Build-time shared map controls, chart cycling and tidal-curve presentation; not a runtime plugin |

The desktop Audio Player is built and distributed by Audio rather than being
another Signal K plugin.

## Ownership rules

- Location Editor is the sole durable owner of location IDs, names,
  classifications and geometry. Tidal Database keys predictions and region
  relationships by those IDs. Weather Database may use a Location as forecast
  context but does not copy the catalogue.
- Tidal Database and Weather Database alone contact their providers and own
  provider caches. Display and Planning do not implement private provider
  access or caches.
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

## Required and optional runtime

Console requires Display, Traffic, Notifications, Audio, Voyages, Navigation
Integrity, Location Editor, Tidal Database and Weather Database. Planning,
Instruments, Vessel Database, Snapshot, Simulator and Pi Controller are
optional, task-specific additions. Derived Data and Charts Provider Simple are
third-party integrations, not AJRM authorities.

## Persistence

Durable state includes plugin configuration, locations and revisions, tidal
and weather caches, tidal definitions, vessel records, routes, voyage bundles,
DR plots, BITE reports and explicit user preferences. Active traffic,
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
port-and-region joins; Planning's three shared services; Snapshot shared-data
evidence; Traffic → Notifications → Audio → Display flow; Capture bundle round
trips; and Navigation Integrity/DR behaviour. Package unit tests remain
responsible for exhaustive internal calculations.

## Safety

The suite assists situational awareness and testing. It is not a certified
navigation, collision-avoidance, anchoring, tide, weather or vessel-control
system, and the skipper remains responsible for decisions.
