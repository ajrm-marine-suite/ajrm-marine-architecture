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
| Alerts | Small read-only alert webapp | queued | Candidate to become a Console view |
| Navigation Reference | Source-aware navigation reference | first pass complete | Retain authority; candidate navigation package module |
| GPS Integrity | Navigation trust provider | queued | Candidate navigation package module |
| DR Plotter | Operational DR webapp | queued | Candidate navigation package webapp |
| Traffic | AIS encounter authority | queued | Retain authority; candidate to own vessel data module |
| Vessel Database | Vessel enrichment store/editor | queued | Candidate Traffic module and view |
| Display | Live chart and traffic display | queued | Retain separately |
| Notifications | Generic notification broker | queued | Retain separately |
| Audio | Audio timeline and delivery | queued | Retain separately |
| Capture | Canonical recorder, replay, voyages | first pass complete | Candidate to absorb Voyage Viewer |
| Voyage Viewer | Read-only voyage analysis | queued | Candidate Capture view/module |
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

The target is twelve active Signal K packages rather than eighteen, subject to
the detailed reviews and migration tests:

1. Console, including the compact read-only Alerts view.
2. Navigation Integrity, containing Navigation Reference, GPS Integrity, and
   DR Plotter as separately testable internal modules and views.
3. Traffic, including the vessel enrichment database and editor.
4. Display.
5. Notifications.
6. Audio.
7. Voyages, containing Capture and Voyage Viewer with one voyage store.
8. Instruments, including instrument alert rules.
9. Harbour Editor.
10. Simulator.
11. Snapshot.
12. Pi Controller.

This preserves different failure and privilege domains. In particular:

- Notifications remains independent of Audio, so visual safety state does not
  depend on speech synthesis or speaker hardware.
- Display remains independent of Traffic, so a renderer does not become the
  collision authority.
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
