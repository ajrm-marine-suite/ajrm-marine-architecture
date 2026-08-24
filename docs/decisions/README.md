# Architecture Decision Records

Decisions are proposed until explicitly accepted.

- [ADR-001: Separate domain meaning, broker mechanics, and audio playback](001-authority-boundaries.md)
- [ADR-002: Runtime notification state does not survive restart](002-runtime-state.md)
- [ADR-003: Audio clients follow one authoritative playback timeline](003-audio-timeline.md)
- [ADR-004: Signal K is the public integration architecture](004-signalk-integration.md)
- [ADR-005: Use Signal K-native observability](005-signalk-observability.md)
- [ADR-006: Version shared contracts](006-contracts-and-migration-modes.md)
- [ADR-007: Compose sailing applications in AJRM Marine Console](007-sailing-console.md)
- [ADR-008: Replace control-heavy alerts clients with a read-only Alerts viewer](008-read-only-alerts-viewer.md)
- [ADR-009: Align recording files, voyage manifests, and file-selection UI](009-recording-file-ownership.md)
- [ADR-010: Use a source-aware own-vessel navigation reference](010-source-aware-navigation-reference.md)
- [ADR-011: Replay physical sensor sources and capture recomputed child voyages](011-sensor-source-replay.md)
- [ADR-012: Standardise map controls around Display and a shared map core](012-shared-map-controls.md)
- [ADR-013: Use Locations as the only spatial catalogue](013-locations-only-contract.md)
- [ADR-014: Separate spatial, tide and weather ownership](014-separate-spatial-tide-weather-ownership.md)
- [ADR-015: Make Marine Planning the tidal-gate definition owner](015-planning-owned-tidal-gates.md)
- [ADR-016: Keep tidal-gate planning data flat](016-flat-planning-tidal-gate-data.md)
- [ADR-017: Resolve Display environment data from a resolved vessel position](017-position-first-display-environment.md)
- [ADR-018: Harden planning and environment contract boundaries](018-harden-planning-environment-contracts.md)
- [ADR-019: Continue collision assessment with bounded explicit estimates](019-bounded-estimated-collision-inputs.md)
