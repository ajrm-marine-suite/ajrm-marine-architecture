# ADR-001: Separate domain meaning, broker mechanics, and audio playback

Status: proposed

## Decision

- Domain providers decide notification meaning.
- AJRM Marine Notifications applies generic live notification mechanics.
- AJRM Marine Audio owns queueing, synthesis, and playback order.
- Clients render projections and invoke explicit actions.

## Consequences

No consumer may infer provider intent from message wording, category, MMSI, or
path name. No provider claims that audio was spoken merely because it requested
sound.
