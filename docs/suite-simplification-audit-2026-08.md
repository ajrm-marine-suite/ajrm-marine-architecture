# AJRM Marine Suite Simplification Audit

Status: package-boundary migrations and retirement cleanup complete
Started: 2026-08-07

## Objective

Review every active Signal K package for correctness, current Signal K plugin
conventions, lifecycle cleanup, API documentation, obsolete compatibility code,
duplicated implementation, and an unnecessarily fine-grained installation
boundary. Current canonical voyages and current suite contracts are the
compatibility baseline. Older voyage formats may be converted externally and
do not justify permanent runtime branches.

## Rules for the cleanup

1. Keep domain authorities explicit even when several authorities share one
   installable package.
2. Prefer standard Signal K paths, notifications, Resources API, Course API,
   subscriptions, base units, access control, and OpenAPI-described plugin APIs.
3. Do not merge components merely because they share a screen or CSS. Merge
   when they share one lifecycle, data owner, persistence owner, or user task.
4. Keep privileged, hardware-dependent, and optional test facilities isolated.
5. Remove a compatibility path only after all current-suite consumers have
   moved to the canonical contract and tests assert that contract.
6. Release small independently testable changes; do not defer safety defects
   until a large merger is complete.

## Current inventory and review state

| Package | Current role | Review state | Initial disposition |
| --- | --- | --- | --- |
| Console | Suite shell, help, BITE | first pass complete | Retain; candidate to absorb Alerts webapp |
| Alerts | Small read-only alert webapp | review complete; retired | Merged into Console in v0.7.0 |
| Navigation Reference | Source-aware navigation reference | merged | Internal GPS Integrity module; standalone package retired |
| GPS Integrity | Navigation trust provider | merger complete | Active combined navigation package |
| DR Plotter | Operational DR webapp | merged | GPS Integrity view; standalone package retired |
| Traffic | AIS encounter authority | first pass complete | Retain separately as the live collision-risk authority |
| Vessel Database | Vessel enrichment store/editor | first pass complete | Retain separately as persistent identity administration |
| Display | Live chart and traffic display | first pass complete | Retain separately |
| Notifications | Generic notification broker | first pass complete | Retain as backend authority; UI moved to Console |
| Audio | Audio timeline and delivery | first pass complete | Retain separately |
| Capture | Recorder, replay, voyages, and review | merger complete | Active combined voyage package |
| Voyage Viewer | Read-only voyage analysis | merged | Capture review view; standalone package retired |
| Instruments | Display and instrument notification provider | review and merger complete | Retain combined package |
| Instrument Alerts | Retirement marker only | retired | Merged into Instruments in v0.8.0 |
| Harbour Editor | Historical harbour editor | retired | Replaced by Location Editor |
| Location Editor | Versioned spatial catalogue and anchoring assistance | active | Sole location/geometry owner |
| Tidal Database | Tide providers, cache, port definitions and region relationships | active | Sole tidal-data owner |
| Weather Database | Weather providers, cache and forecast resolution | active | Sole weather-data owner |
| Marine Planning | Gate and anchor planning consumer | active | Read-only consumer of shared services |
| Simulator | Optional test data source | first pass complete | Retain separately |
| Snapshot | Optional diagnostic evidence provider | first pass complete | Retain separately |
| Pi Controller | Privileged Pi operations | first pass complete | Retain separately |
| Logger | Retirement marker only | retired | Remove from active suite/install model |

`@ajrm-marine/map-core` is a reviewed shared browser library, not a Signal K
plugin. It continues to standardise map controls without merging the four
different map applications. AJRM Marine Audio Player is a separate desktop
client, not a Signal K plugin or installation boundary. Its canonical source
is the `desktop-player` directory in the AJRM Marine Audio repository so the
server contract and desktop packages cannot drift across duplicate repos.

## Current installation boundaries

The reviewed suite now has sixteen active Signal K packages:

1. Console, including the compact read-only Alerts view.
2. Navigation Integrity, containing Navigation Reference, GPS Integrity, and
   DR Plotter as separately testable internal modules and views.
3. Traffic.
4. Vessel Database.
5. Display.
6. Notifications.
7. Audio.
8. Voyages, containing recording, replay and review with one voyage store.
9. Instruments, including instrument alert rules.
10. Location Editor.
11. Tidal Database.
12. Weather Database.
13. Marine Planning.
14. Simulator.
15. Snapshot.
16. Pi Controller.

This preserves different failure and privilege domains. In particular:

- Notifications remains independent of Audio, so visual safety state does not
  depend on speech synthesis or speaker hardware.
- Display remains independent of Traffic, so a renderer does not become the
  collision authority.
- Vessel Database remains independent of Traffic: persistent vessel identity
  enrichment and bulk administration have a different lifecycle and failure
  mode from live CPA/TCPA and collision-risk evaluation.
- Location Editor remains the independent durable spatial provider. Tidal and
  Weather Databases own their provider data and caches; Planning consumes them.
- Pi Controller remains isolated because it performs privileged host actions.
- Simulator remains optional and removable from a sailing installation.

Both boundary migrations are complete. GPS Integrity contains Navigation
Reference and DR Plotter, while Capture contains voyage review. Their internal
status projections remain versioned component contracts; they no longer imply
separately installable plugins.

Traffic and Vessel Database explicitly remain separate. Traffic owns transient
AIS encounter and collision-risk state; Vessel Database owns durable identity,
classification and manual enrichment. Combining them would couple safety
calculation availability to an administrative data store and make both harder
to test and recover.

## Findings and changes

### Navigation Reference

- The code and test description disagreed: the resolver listened to the retired
  `plugins.ajrmMarineLogger.playback` path while Capture publishes
  `plugins.ajrmMarineCapture.playback`.
- Version 0.1.6 now follows the canonical Capture clock and has no runtime
  dependency on Logger.

### Capture

- Capture is the current canonical recording and replay authority. The
  architecture and several names still reflected the former two-plugin design.
- Current-format completion checkpoints were being tested with Logger-era
  result-segment fields during restart recovery. Version 0.7.16 validates the
  canonical replay result and durable recomputed output instead.
- Signal K `stop()` now awaits active-voyage preservation rather than starting
  asynchronous finalisation and returning immediately.
- Version 0.8.0 removes the legacy converter, reference-mode portable-download
  rebuilding, dead Logger API/HTTP fallbacks, capture-segment copying, old raw
  capture indexing and old-directory discovery. The packaged implementation is
  about 28 percent smaller (193.7 kB to 139.6 kB for `plugin/index.js`).
- Version 0.8.0 also adds an OpenAPI document for every registered HTTP route.

### Console

- Version 0.6.23 removes three unreachable Logger BITE implementations, the
  retired Logger API registry, Logger test fixtures, and the obsolete Capture
  file-mode assertion (314 lines removed).
- Console retains one intentional retirement filter so an accidentally
  installed Logger marker is not presented as an active application.
- Console already describes its HTTP routes with OpenAPI. Its larger BITE file
  remains a candidate for decomposition into domain-focused modules during the
  full Console review.
- Version 0.7.0 absorbs the read-only Alerts view and its three display
  preferences. Notifications remains the sole alert-state authority and Audio
  remains the sole speech/delivery authority. The former Alerts package is now
  a disabled retirement marker, removing one active plugin and one duplicate
  webapp lifecycle without combining safety authorities.

### Voyage Viewer

- Version 0.7.0 makes current self-contained Capture bundles the only runtime
  input and removes raw-log, Clips, embedded-segment, external-Logger and old
  plot-cache fallbacks (net 280 lines removed across the release).
- Its generic file-kind HTTP surface is now a voyage-only API with a complete
  OpenAPI description.
- The recomputation review was still validating the retired Logger-era result
  segment manifest. It now validates Capture's current replay-result contract,
  explicit timing validity, durable checkpoint, recomputed-output contract and
  exact embedded byte count.
- Capture and Viewer now agree on one store and one bundle contract. Their
  remaining cross-package API is only download preparation, making them ready
  for a later package-boundary merge without first mixing compatibility logic
  into that migration.

### Notifications

- Version 0.7.0 remains an independent backend authority, but removes its
  redundant diagnostics webapp. Console Alerts is now the single normal alert
  interface; BITE and the documented HTTP API retain diagnostic access.
- Active-envelope expiry now schedules its own publication. Previously an
  expired alert could remain visible indefinitely until another notification
  or HTTP status request happened.
- Whole-tree Signal K notification deltas now preserve nested `null` leaves,
  so standard clears are not lost while flattening the tree.
- Stale active/resolved envelopes and duplicate events are rejected before
  supersession, source remapping or Audio delivery. Expiry no longer makes an
  unrelated rejected event audible.
- `start()` and `stop()` now own subscriptions and the expiry timer
  idempotently, following the Signal K plugin lifecycle contract.
- Unused saved-state serialization and the unused provider `broker: false`
  compatibility branch were removed; broker state is explicitly runtime-only.

### Audio

- Version 0.7.0 remains separate from Notifications because speech synthesis,
  child processes, sound hardware and radio-stream clients form a distinct
  failure domain from visual notification state.
- Both the ordinary plugin router and Signal K API command routes now enforce
  write access, and every current route is described by OpenAPI.
- Asynchronous `stop()` now prevents new queue work, terminates active Piper,
  FFmpeg and local-player processes, closes stream clients, and waits for the
  public stream server to close.
- Interrupted, superseded and failed preparations now remove temporary and
  partially published audio artefacts. Only successfully delivered output is
  retained.
- The retired AIS Plus audio directory fallback and `aisPlusMuted` status alias
  were removed. The current AJRM Marine directory and Traffic Audio Policy are
  the only runtime contracts.
- Repeated executable and voice scans during each status build were reduced by
  sharing one dependency result.

### GPS Integrity

- Version 0.7.0 preserves GPS Integrity as the navigation-trust authority while
  preparing it to become an internal module of the proposed Navigation
  Integrity package.
- Settings, baseline reset and manual-fix routes now require Signal K write
  access and all current routes have an OpenAPI description.
- Partial settings updates previously replaced omitted values with defaults,
  which could silently re-enable alerts or reset the independent-DR realignment
  interval. Omitted settings are now preserved.
- Cached samples are cleared on lifecycle restart, evaluation timers no longer
  hold the Node process open, and running state is explicit.
- The unused flat `deadReckoning.*` Signal K projection was removed. The current
  suite uses the unambiguous `deadReckoning.operational.*` and
  `deadReckoning.integrity.*` paths. The full integrity state still temporarily
  carries its older internal `deadReckoning` member until remaining consumer
  fallbacks are removed as part of the Navigation Integrity merger.

### DR Plotter

- Version 0.7.0 confirms that DR Plotter belongs in the proposed Navigation
  Integrity package: it requires GPS Integrity, renders that authority's exact
  state, and sends observed fixes back to GPS Integrity. It does not own a
  separate navigation decision.
- All track, plot-fix and settings mutations now require Signal K write access;
  every current route has an OpenAPI description.
- `stop()` now waits for in-flight navigation-state and plot-fix persistence,
  and clears its published status projection. Disabled/stopped instances no
  longer record startup or subscription state.
- Status and documentation claimed that the private `plot-fixes.json` and
  `operational-track.json` files were copied into Capture bundles. Current
  Capture v0.8 does not read those files; the false `capturePath`/`captureFile`
  contract was removed.
- The operational DR state track is already recorded canonically by Capture.
  Navigator plot fixes still need an explicit current voyage handoff when the
  Navigation Integrity and Voyages package boundaries are implemented; this
  must be a versioned API/module contract, not direct private-file coupling.

### Instruments and Instrument Alerts

- Instruments v0.7.0 and Instrument Alerts v0.7.0 establish reviewed,
  independently releasable baselines before the package merger. Both now
  declare Node 20 and OpenAPI; both own restart/stop state explicitly.
- Instruments safely replaces subscriptions on restart and no longer accepts
  the obsolete `engineRoomTemperaturePath` configuration alias. Its custom
  pilot-helm and signed-XTE paths remain intentional current-suite contracts:
  they gate ambiguous raw tiller-pilot data and give the alert engine one
  nullable, unit-defined input.
- Instrument Alerts write operations now require Signal K read/write or admin
  access. Its notification output remains standard `notifications.*` state,
  method and message; the richer AJRM envelope remains optional data rather
  than replacing the Signal K contract.
- The one-time migration from the pre-suite
  `audible-instruments-settings.json` name was removed. The current runtime
  settings file and monitor contract remain unchanged.
- The merger should retain the monitor evaluator as an internal provider module
  and expose its editor as an Instruments view. Display rendering must not own
  threshold decisions, and the standard notification paths must remain useful
  when Notifications or Audio is absent.
- Instruments v0.8.0 completes that merger. The evaluator, AJRM notification
  envelope adapter, settings persistence, and editor moved unchanged behind
  `/alerts/*` routes and `alerts.html`; the main status includes provider state
  and the package has one combined 42-test suite.
- The former Instrument Alerts v0.8.0 package is now a disabled inert retirement
  marker. Its implementation and duplicate webapp were removed. Updating or
  uninstalling it prevents two providers from announcing the same thresholds.

### Traffic

- Traffic v0.7.0 remains a separate safety authority. It owns AIS observation
  qualification, CPA/TCPA, encounter lifecycle, wording, profiles, silencing,
  and standard collision notifications; no Vessel Database persistence or UI
  was merged into it.
- Restart now replaces both Subscription Manager subscriptions, resets all
  session-scoped publication caches, and invalidates outstanding harbour-region
  loads. Post-stop deltas, calculations, and region refreshes are ignored.
- Stop clears active notifications and all six retained
  `plugins.ajrmMarineTraffic.*` projections, rather than leaving clients with a
  stale authoritative session after the provider is disabled.
- Every profile, Auto Profile, audio-policy, and target-silence mutation now
  requires Signal K read/write or admin access. OpenAPI now describes every
  registered route and both supported PUT/POST command forms.
- Obsolete persisted `muted` and `manualMute` placeholders were removed. Mute
  remains intentionally runtime-only. Capture terminology replaces the last
  Logger names in current replay tests and documentation.
- The collision calculation and notification policy remained unchanged and its
  complete 155-test suite passed, including SAR-aircraft exclusion, hovercraft
  inclusion, source-qualified AIS class, timing/freshness, downgrade holds,
  COLREG disclaimer constraints, and replay warm-up suppression.

### Vessel Database

- Vessel Database v0.8.0 remains separate from Traffic. It is the durable MMSI
  identity and static-particulars authority; Traffic remains the live
  observation, encounter, CPA/TCPA, and collision-notification authority.
- All import, online-lookup, deletion and test-cleanup mutations now require
  Signal K read/write or administrator access. OpenAPI describes every current
  route, including cancellation and bulk deletion.
- Restart now owns exactly one raw delta listener, resets session statistics,
  and cancels an in-flight ITU MARS lookup. Abort propagation prevents a remote
  response from saving data, publishing status, or changing plugin state after
  shutdown.
- The raw all-context delta listener is retained intentionally: static AIS
  discovery needs vessel context, source, timestamp and grouped update values;
  a bounded self-vessel subscription cannot provide that contract.
- The duplicate bulk-delete route and one-time obsolete dimension scrub were
  removed. Generic `design.dimensions` remains ignored, while only current
  standard AIS antenna offsets are learned and filled.
- Stop retracts the retained summary only when summary publication is enabled.
  The reviewed package passes 27 tests, including lifecycle, access-control,
  OpenAPI parity, import validation, SAR-aircraft classification and exact-MMSI
  test cleanup.

### Display

- Display v0.7.0 remains an independent renderer and operational webapp.
  Traffic remains the live collision-risk authority and Vessel Database remains
  the separate durable identity/classification authority.
- The webapp still called three Display endpoints that no longer existed:
  Audio sound check, Notifications history clearing and Traffic all-well
  policy. Those controls now call their authoritative providers directly.
- Every Display mutation now requires Signal K read/write or administrator
  access, and OpenAPI describes the exact 25-route HTTP surface. Route-parity
  and unauthenticated-write tests prevent drift.
- Stop retracts `plugins.ajrmMarineDisplay`, removes both in-process API
  registrations and invalidates delayed route initialization. Browser speech
  is projected only when Notifications explicitly selects audio delivery.
- The nonexistent GPS-warning pause action now honestly dismisses the local
  popup; system sound policy remains with Traffic and Audio. Dead browser
  announcement logging and an unimplemented CPA-shape checkbox were removed.
- A dormant 553-line local CPA/risk calculator was deleted. Production Display
  imported only its angle conversion functions, so keeping a second safety
  engine contradicted Traffic ownership and created needless maintenance risk.
  The replacement angle utility is seven lines.
- The package dropped unused Lodash and obsolete `AisPlus` internal naming.
  All 356 tests, production dependency audit, build, scaffold validation,
  package dry-run and diff checks passed.

### Locations replaces Harbour Editor

- Location Editor is the sole durable location catalogue. Harbour Editor and
  Signal K `Harbour:` region mirroring are retired.
- Automatic profile areas are explicit versioned Location records. Traffic and
  Display consume them directly through `ajrm-marine-locations-service-v1`;
  Snapshot records the same projection.
- Location imports accept only the versioned Locations catalogue contract.
  There is no runtime prefix matching, dual-write path or Harbour Editor
  compatibility provider.
- Marine Planning consumes Locations, tide and weather services directly and
  is exposed by Console as a current suite app.

### Simulator

- Simulator v0.8.0 remains a separate optional home-test provider. It owns one
  coordinated own-vessel, environment, GNSS and synthetic AIS model; it does
  not absorb Traffic or Vessel Database responsibilities.
- All 15 state-changing HTTP routes now require Signal K read/write or
  administrator access, and OpenAPI 3.1 describes the exact 16-route API.
- The plugin publishes a versioned retained `plugins.ajrmMarineSimulator`
  lifecycle projection so Console/BITE can verify output state without relying
  on prose status. Stop emits the final quiet sample, clears stale environment
  and target values, retracts that projection and releases its timer.
- Runtime persistence now covers the editable battery, transducer offset and
  variation parameters that were previously lost at restart. Direction
  controls reject invalid values instead of silently treating them as right/up.
- The obsolete `/own/autopilot` alias, duplicated-GPX importer repair and three
  unused value builders were removed. Runtime settings advance to v2 and
  intentionally ignore the old route-repair-era settings once.
- All 58 tests, production dependency audit, package dry-run and diff checks
  passed.

### Snapshot

- Snapshot v0.7.0 remains a small standalone diagnostic provider because both
  Capture and Console consume its in-process API. Merging it into either
  consumer would invert the other dependency and duplicate snapshot policy.
- Restart releases the prior subscriptions and API registrations before
  creating a new session. Plugin version/readiness now use Signal K's standard
  plugin status surface.
- The suite summary reads collision profiles, Auto Profile, audio policy and
  targets directly from Traffic. Display remains the source only for rendered
  alert/announcement history and harbour geometry.
- Retired Logger, Companion, old announcer and split Alerts/Instrument Alerts
  compatibility were removed from options, UI, diagnostics and package
  discovery. Remote access remains explicit opt-in and now rejects an
  explicitly unauthenticated Signal K request.
- Local diagnostic HTTP responses are bounded to 2 MiB. OpenAPI matches both
  read-only routes. All 25 tests, production dependency audit, package dry-run
  and diff checks passed.

### Pi Controller

- Pi Controller v0.7.0 remains separate because it is the only package that
  owns host-level health, shutdown/reboot, support installation and SD-card
  backup operations. These privileges do not belong in a navigation or display
  plugin.
- All four host-action routes now require Signal K read/write or administrator
  access in addition to their existing enable and explicit-confirmation gates.
  OpenAPI 3.1 describes the exact five-route surface.
- Restart clears both monitoring timers, pending power timers and owned support
  or backup child processes. Late telemetry, shutdown-watch and child callbacks
  are generation-guarded; stop retracts retained telemetry.
- Concurrent power and support actions are rejected. Disk monitor paths are
  passed after `df --`, detached spawn failures are handled, and the package
  now declares the suite Node.js 20 baseline.
- Syntax, shell syntax, three access/contract tests, production dependency
  audit, package dry-run and diff checks passed.

### Map Core

- Map Core v0.7.0 remains the shared browser implementation for chart
  selection, Charts Provider Simple folder controls, overlapping-chart cycling,
  coordinate formatting, control icons and map-panel layout. Display, DR
  Plotter, Voyage Viewer and Harbour Editor retain only their domain layers and
  actions.
- The chart selector now unregisters its window resize, DOM change and Leaflet
  map listeners when removed. Event-target validation no longer assumes a
  browser-global `HTMLInputElement`, which also makes the shared control easier
  to test and embed.
- Display, DR Plotter, Harbour Editor and Voyage Viewer v0.7.1 pin and bundle
  the reviewed Map Core v0.7.0 release. Voyage Viewer also moves from Node.js
  18 to the suite Node.js 20 baseline.
- Signal K packaging validation then identified package-relative App Store icon
  paths in Display, DR Plotter and Harbour Editor. Their v0.7.2 releases point
  at the icons actually included beneath `public/`.
- All 12 tests, package dry-run and diff checks passed. The library has no
  production dependencies or HTTP surface.

### Audio Player

- Audio Player v0.7.0 remains a standalone Electron client for the Audio
  plugin, not a Signal K plugin. It does not add another server-side authority
  or installation boundary.
- Polls can no longer overlap or update the interface after disconnect. Muting
  pauses the current announcement and unmuting resumes it instead of leaving
  playback stuck.
- Electron sandboxing, external-window/navigation denial and a content security
  policy now constrain the renderer. Private-host certificate exceptions use
  validated IP addresses rather than permissive digit patterns.
- Syntax tests, host-policy tests, production dependency audit, package dry-run
  and diff checks passed. This local-only desktop repository has no configured
  Git remote; v0.7.0 is committed and tagged locally rather than published.

## Signal K conformance backlog

- Add or complete `getOpenApi()` for every package that registers HTTP routes.
- Review read-only versus administrative route access explicitly.
- Prefer Subscription Manager for bounded path subscriptions; retain raw delta
  listeners only where source-preserving capture or all-context discovery
  genuinely requires them.
- Verify every `stop()` removes listeners, subscriptions, timers, file handles,
  child processes, and in-process API registrations, and returns a Promise when
  cleanup is asynchronous.
- Keep standard Signal K units internally and standard notification/resources
  surfaces independently useful without AJRM extensions.
- Remove direct private filesystem/API coupling where a versioned Signal K or
  in-process contract exists.
