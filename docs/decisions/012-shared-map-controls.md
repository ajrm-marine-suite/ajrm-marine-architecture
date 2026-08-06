# ADR-012: Standardise map controls around Display and a shared map core

Status: accepted

Date: 2026-08-06

## Context

AJRM Marine Display, DR Plotter, Voyage Viewer and Harbour Editor all render
Leaflet maps and Signal K chart resources. Their chart selectors evolved
separately, so the same basemap, OpenSeaMap, Auto Charts and overlapping-chart
tasks appeared in different places and behaved differently. Changes to chart
ranking or Charts Provider Simple folder controls would otherwise need four
independent implementations.

The applications still have different jobs. Display presents live vessel and
route state; DR Plotter presents navigator fixes and dead reckoning; Voyage
Viewer reviews completed recordings; Harbour Editor edits regions. A single
large map application or a shared server plugin would therefore merge concerns
that should remain separate.

## Decision

AJRM Marine Display is the reference design for common map interaction. A
small, versioned, browser-side package, `@ajrm-marine/map-core`, owns the common
contract `ajrm-marine-map-shell-v1` and reusable implementation for:

- chart-resource normalisation and bounds handling;
- native-zoom, overzoom and overlap ranking;
- basemap and overlay selection;
- nested Charts Provider Simple folder enable/disable controls;
- cycling overlapping charts and returning to automatic selection; and
- a common Display-style action-button toolbar for application-owned tools;
- coordinate-format helpers.

Display retains its mature local presentation and consumes the core ranking
contract. DR Plotter, Voyage Viewer and Harbour Editor use the shared selector,
cycle and action-toolbar controls with Display-compatible labels and order.
All map interaction controls occupy the upper-left Leaflet stack: the native
`+ / −` zoom control is first, followed by chart controls and then
application-specific actions. Action buttons form a vertical stack. Left-side
application drawers start closed and open to the right of the toolbar so they
do not obscure those controls. App-specific behaviour and overlays remain in
their owning applications.

The map core is an internal build/development dependency, not a separately
installed Signal K plugin. Apps that do not bundle dependencies at runtime
vendor the exact tagged core module and stylesheet into their published web
assets, while recording the same tag in `devDependencies`. Browser asset URLs
include the app and core versions so an upgrade does not retain stale controls.

## Consequences

- The four map apps give users one chart-selection model without becoming one
  monolithic application.
- Auto Charts ranking and overlapping-chart behaviour have one tested owner.
- Charts Provider Simple folder hierarchy and inherited disabling are exposed
  consistently.
- Zoom, chart selection and application actions use the same left-side visual
  hierarchy in every map app.
- A change to the common contract requires a tagged map-core release followed
  by explicit consumer patch releases; consumers do not float on an untagged
  branch.
- Display remains the place to prove a new map interaction before promoting it
  into the shared contract.
- App-specific map tools are shared only when their meaning applies across the
  suite; identical placement is not forced for controls that would be
  misleading in another app.

## Verification

Each consumer must pass its normal test and package dry run. The vendored core
version must match the pinned development dependency, and the map page must
load the core stylesheet before the app stylesheet. Map/chart changes remain
subject to browser-level smoke testing on the Signal K target before broader
rollout.
