# ADR-018: Harden planning and environment contract boundaries

Status: accepted  
Date: 2026-08-23

## Context

The move to Planning-owned tidal-gate rows and position-first environment
presentation established the correct ownership boundaries, but the first
implementation left several boundary checks implicit:

- Tidal Database removed gate operations while retaining its old v1 service,
  status and diagnostics identities;
- Location mutations protected a gate Location used by Planning, but not the
  standard-port Location referenced by that gate row;
- Planning trusted a valid tide response without proving that its selected port
  was the exact requested reference port;
- Display could treat an aged own-position sample as fresh and its programmatic
  startup recenter could be mistaken for a manual pan; and
- concurrent Weather Database requests could contend for one temporary cache
  filename, while fallback presentation described any cached contributing
  provider as if the primary forecast itself were cached.

These are contract and coordination defects rather than reasons to merge the
domain owners again.

## Decision

Tidal Database advances its in-process service, published status and diagnostic
snapshot to major version 2. The v2 identity describes the provider, port,
region, cache and tide-resolution surface after removal of tidal-gate methods.
Consumers reject unsupported major versions; a removed member is never hidden
behind an unchanged contract identity.

Location Editor remains the only authority for Location names and types. Tidal
Database joins port and region definitions to the exact Location UUID and uses
the Location-owned name for presentation. Its durable definitions advance to
version 3 and call the non-authoritative fallback label
`cachedLocationName`. Service projections expose `nameSource` as `location` or
`cached`. The fallback must not be editable as a second authoritative name, and
a write must not proceed when the Location ID or required type cannot be
verified. One Location cannot be both a tidal standard port and tidal secondary
port. A correction port is operational only while every parent join in its
chain is valid and the chain terminates at a provider-backed standard port.

If retired gate data is still awaiting Planning acknowledgement, Tidal
Database serializes later name-cache and definition edits as a genuine legacy
v1 file with legacy `name` fields. The gate-free v3 durable rewrite happens
only after Planning has persisted, verified and acknowledged the migration, so
rollback to the prior reader remains possible during the coordinated cutover.

Planning's Location-mutation coordination covers both sides of every live row:
the gate UUID must remain classified `tidalGate`, and its
`referencePortLocationId` must remain classified `tidalStandardPort`. Public
Location mutations, catalogue replacement/merge and shared classification
removal all enter the same coordinator. The guard is evaluated while Planning's
row-mutation queue is held so neither owner can validate against an already
superseded join.

Planning accepts a tide resolver result only when its explicit contract is
supported and `selectedPort.id` exactly equals the requested
`referencePortLocationId`. A missing or different ID is an unavailable join,
not permission to relabel spatially selected events. The response includes the
reference levels used with those events so a manual refresh cannot combine
current events with an earlier catalogue snapshot.

Display classifies own-position freshness from explicit sample time and a
30-second own-position freshness limit. Missing age evidence fails closed. An
older usable coordinate is retained as last-known rather than reclassified from
target-age policy. Programmatic startup centring and resize events without an
operator `movestart` preserve vessel follow; an actual operator pan still
disables follow. Automatic chart selection is updated for the resolved vessel
position before the startup cover is removed.

Weather Database serializes or coalesces simultaneous provider work for the
same cache key and uses a unique atomic-replacement temporary file. Fallback
metadata describes the selected primary forecast. A cached secondary source
that only fills otherwise-null fields does not make a live primary forecast a
nearest-cached forecast. Read routes register at Signal K's `readonly` scope;
force-refresh registers at `readwrite` and retains a handler-level read/write or
administrator check. Planning exposes cache-aware gate weather and tide reads
separately from its `readwrite`-protected manual-refresh POST routes, so it does
not create an unauthenticated path around either provider owner.

## Consequences

- Mixed versions fail with an explicit unsupported contract instead of
  appearing compatible and failing on a missing method.
- Location edits cannot silently orphan either side of a Planning row.
- Contradictory port roles and broken correction-parent chains cannot remain
  operational.
- Pending gate migration remains rollback-readable until acknowledgement.
- Fetched tide rows and calculation levels have one verifiable reference-port
  identity.
- A stale GPS coordinate remains available without being presented as a fresh
  fix or enabling fresh-position-only anchoring assistance.
- Programmatic initial centring does not disable normal vessel following.
- Concurrent weather clients share one successful provider result and cache
  write, and the displayed fallback wording matches the forecast actually
  selected.

This decision hardens, rather than replaces, the ownership and workflow in
[ADR-016](016-flat-planning-tidal-gate-data.md) and the position-first behavior
in [ADR-017](017-position-first-display-environment.md).
