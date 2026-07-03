# ADR-002: Runtime notification state does not survive restart

Status: proposed

## Decision

Broker active state, recent activity, audio requests, and playback queues are
memory-only and scoped to a process session.

## Consequences

Providers republish current conditions after startup. Clients use session IDs to
discard stale projections. Configuration may persist; live alert state does not.
