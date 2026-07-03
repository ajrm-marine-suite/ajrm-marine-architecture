# ADR-007: Compose sailing applications in AJRM Marine Console

Status: accepted  
Date: 2026-06-20

## Decision

Create `signalk-ajrm-marine-console` as the preferred sailing workspace while
keeping Traffic, Notifications, Audio, Display, Alerts, and Instruments as
independently installable Signal K plugins and webapps.

The first Console release uses a small manifest and same-origin iframe tabs for
the proven existing webapps. It adds only shell-owned concerns:

- operational navigation;
- tab visibility and default-tab preferences;
- suite availability summary;
- a constrained AJRM Marine Capture incident-recording action.

The Console does not host simulators, Harbour Editor, Snapshot, Capture
playback/file management, or deep administration.

## Rationale

The suite has repeated presentation surfaces for targets, alerts, recent
activity, profiles, sensitivity, silence, mute, health, authentication and
responsive navigation. Most safety and delivery policy is no longer duplicated;
the remaining duplication is chiefly client composition and rendering.

Browser tabs provide isolation but poor onboard navigation. A fully general
runtime JavaScript plugin system would introduce premature complexity around
CSS isolation, version compatibility, authentication and failure containment.
The Console shell provides integrated navigation now while preserving strong
component boundaries.

## Evolution

Embedded modules may be replaced incrementally by native Console panels that
consume published contracts. Specialist webapps remain available standalone.
Third-party runtime module loading is deferred until there is a demonstrated
need beyond the maintained suite.

## Constraints

- Backend authority remains with the owning plugins.
- Console modules must not recalculate safety state or audio policy.
- A failed or absent module must not prevent other tabs from operating.
- Chart interaction remains browser-local and must not wait for Console status.
- Incident capture may start and stop recording only; replay remains outside
  the sailing Console.
