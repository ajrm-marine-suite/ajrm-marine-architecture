# ADR-005: Use Signal K-native observability

Status: proposed

## Decision

Suite plugins use Signal K's `app.debug()` and `app.error()` facilities so
diagnostics are controlled and viewed through the Signal K Server Log.

Investigations combine:

- raw Signal K data logging for incoming interface evidence;
- normalized delta capture/replay for Signal K paths and plugin output;
- a History API implementation for longer-term trends where needed;
- correlated provider, broker, Audio, and client lifecycle records.

## Consequences

- Plugins log transitions and failures, not every refresh or sensor sample.
- High-frequency values are captured as data rather than duplicated in text.
- Session, correlation, event, request, and subject identifiers connect logs to
  captured deltas.
- Debug can be enabled per plugin through Signal K and remains quiet by default.
- Credentials and private tokens are never logged.
- Plugin-specific diagnostic pages summarize Signal K-observable evidence; they
  do not become separate authoritative logging databases.
