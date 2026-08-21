# ADR-014: Separate spatial, tide and weather ownership

Status: accepted  
Date: 2026-08-21

## Decision

Location Editor is the sole durable spatial catalogue. It owns stable IDs,
names, types, geometry, revision history, anchoring attributes and automatic
profile areas.

Tidal Database owns provider configuration, station mappings, prediction
caches, secondary-port corrections, tidal-region serving-port relationships
and tide calculations. Weather Database independently owns weather providers,
provider-separated caches and forecast aggregation. Both refer to Locations by
stable ID and must not become competing spatial catalogues.

Display and Marine Planning consume those services. They do not fetch or cache
provider data themselves. Location Editor may present joined tidal-region
controls, but writes the relationship through Tidal Database rather than
storing it in Location properties.

## Consequences

- Tide and weather data remain usable offline from their own durable caches.
- A Location rename or geometry edit has one owner.
- Provider-specific details do not leak into Display or Planning.
- Runtime BITE must validate the Location/Tidal joins and the consumer service
  contracts, not merely that each plugin is installed.
- Harbour Editor and Location-owned tide/weather services are not supported
  transition models.
