# ADR-003: Audio clients follow one authoritative playback timeline

Status: proposed

## Decision

AJRM Marine Audio owns the only semantic audio queue. It synthesizes once and
publishes playback lifecycle events plus the selected rendered asset. The
Signal K server speaker, Alerts clients, desktop players, and stream outputs
are sinks of that timeline.

## Consequences

Alerts and desktop players do not independently reorder or infer
announcements. A short scheduled-start window may synchronize open clients with
the Signal K server speaker, but remote readiness must not impose a long delay
on the local speaker.
