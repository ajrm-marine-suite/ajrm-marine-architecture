# ADR-017: Resolve Display environment data from a resolved vessel position

Status: accepted  
Date: 2026-08-23

## Decision

Display does not reveal its initial chart, automatic tide result or weather
forecast until it has resolved the position on which the initial chart will be
centred. A fresh, valid own-vessel position is preferred. If GPS is lost or no
fresh fix is available, a retained last-known fix is accepted and the chart is
centred there before it is revealed. After a bounded 15-second wait with no
usable fresh or retained position, Display may reveal a usable no-GPS chart; it
must continue listening for a usable fix and must not treat the chart centre as
the vessel's location.

Automatic tide and weather selection both use that resolved fresh or
last-known position, but they remain separate resolutions. They are not started
until the chart has been centred. Tidal Database selects the tidal port.
Weather Database selects the nearest eligible Location independently, so the
weather Location need not be the tidal Location. An explicit tidal-port choice
may still be used without automatic position selection.

Weather Database, not Display, owns offline forecast fallback. Its additive
`resolveNearest` service operation and `/weather/nearest` HTTP endpoint first
select the nearest eligible Location. A recent exact-location cache is reused;
otherwise Weather Database attempts the provider. If that attempt fails it may
use a non-expired exact-location cache, and only after that failed provider
attempt may it select a different nearest non-expired cached coordinate group.
The provider attempt has a bounded deadline so a blackholed connection cannot
prevent offline fallback. Provider records from different places are never
combined. It returns
`ajrm-marine-weather-location-resolution-v1` metadata containing the requested
position, selected Location or cached position, selection mode, cache-fallback
state, reason and great-circle distance in metres.

Display renders the returned hourly weather table, provenance, freshness and
distance. It neither scans weather-cache files nor contacts a weather provider.

## Consequences

- Startup no longer flashes a chart or tide result for the configured fallback
  centre before moving to the vessel.
- Loss of GPS retains an operational last-known chart centre and permits tide
  and weather context at that explicitly stale position.
- A tide port and a weather Location may legitimately have different names and
  coordinates.
- An offline weather report is visibly identified with its distance from the
  resolved fresh or last-known vessel position rather than presented as if it
  were local.
- The existing `ajrm-marine-weather-database-service-v1` contract remains
  compatible: `resolveNearest` and location-resolution metadata are additive.
- A browser with neither a fresh nor retained position remains usable after the
  bounded wait, but automatic environment selection remains unavailable until
  a usable position arrives.
