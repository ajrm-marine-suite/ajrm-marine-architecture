# ADR 013: Locations is the only place and profile-area contract

## Status

Accepted, 2026-08-19.

## Decision

AJRM Marine Location Editor owns the only durable catalogue for harbours,
marinas, anchorages, moorings, tidal locations, gates, hazards and other marine
places. Automatic Harbour-profile polygons are marked by
`properties.automaticProfileArea` and supplied through
`ajrm-marine-locations-service-v1.profileAreas()`.

Traffic, Display, Snapshot and Marine Planning consume Locations directly.
They must not discover places through Signal K `regions`, a `Harbour:` name
prefix, or a Harbour Editor compatibility API. Console treats Location Editor
as required, exposes Marine Planning, and BITE verifies the shared contracts and
consumer counts.

Harbour Editor is retired and hidden by Console. Existing installations remove
the package after installing the Locations-only releases.

## Consequences

- One location is edited, versioned, exported, merged and deleted in one place.
- Profile switching and diagnostics cannot diverge between duplicate stores.
- Old Harbour Editor exports and `Harbour:` regions are not accepted at runtime.
- A bounded on-read field rename may preserve an already-versioned Location
  catalogue, but no legacy provider, endpoint or dual-write behaviour remains.
