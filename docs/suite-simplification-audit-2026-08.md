# AJRM Marine Suite Simplification Audit

Status: active review  
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
| Navigation Reference | Source-aware navigation reference | first pass complete | Retain authority; candidate navigation package module |
| GPS Integrity | Navigation trust provider | first pass complete | Candidate navigation package module |
| DR Plotter | Operational DR webapp | queued | Candidate navigation package webapp |
| Traffic | AIS encounter authority | queued | Retain separately as the live collision-risk authority |
| Vessel Database | Vessel enrichment store/editor | queued | Retain separately as persistent identity administration |
| Display | Live chart and traffic display | queued | Retain separately |
| Notifications | Generic notification broker | first pass complete | Retain as backend authority; UI moved to Console |
| Audio | Audio timeline and delivery | first pass complete | Retain separately |
| Capture | Canonical recorder, replay, voyages | first pass complete | Candidate to absorb Voyage Viewer |
| Voyage Viewer | Read-only voyage analysis | first pass complete | Candidate Capture view/module |
| Instruments | Configurable instrument display | queued | Candidate to absorb Instrument Alerts |
| Instrument Alerts | Instrument notification provider | queued | Candidate Instruments module |
| Harbour Editor | Harbour resource editor/provider | queued | Retain separately |
| Simulator | Optional test data source | queued | Retain separately |
| Snapshot | Optional diagnostic evidence provider | queued | Retain separately |
| Pi Controller | Privileged Pi operations | queued | Retain separately |
| Logger | Retirement marker only | retired | Remove from active suite/install model |

`@ajrm-marine/map-core` is a shared browser library, not a Signal K plugin. It
continues to standardise map controls without merging the four different map
applications.

## Proposed target installation boundaries

The target is thirteen active Signal K packages rather than eighteen, subject to
the detailed reviews and migration tests:

1. Console, including the compact read-only Alerts view.
2. Navigation Integrity, containing Navigation Reference, GPS Integrity, and
   DR Plotter as separately testable internal modules and views.
3. Traffic.
4. Vessel Database.
5. Display.
6. Notifications.
7. Audio.
8. Voyages, containing Capture and Voyage Viewer with one voyage store.
9. Instruments, including instrument alert rules.
10. Harbour Editor.
11. Simulator.
12. Snapshot.
13. Pi Controller.

This preserves different failure and privilege domains. In particular:

- Notifications remains independent of Audio, so visual safety state does not
  depend on speech synthesis or speaker hardware.
- Display remains independent of Traffic, so a renderer does not become the
  collision authority.
- Vessel Database remains independent of Traffic: persistent vessel identity
  enrichment and bulk administration have a different lifecycle and failure
  mode from live CPA/TCPA and collision-risk evaluation.
- Harbour Editor remains an independent durable-resource provider.
- Pi Controller remains isolated because it performs privileged host actions.
- Simulator remains optional and removable from a sailing installation.

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
