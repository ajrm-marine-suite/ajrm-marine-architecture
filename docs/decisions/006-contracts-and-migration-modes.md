# ADR-006: Version shared contracts

Status: accepted, updated after AJRM Marine cutover.

## Decision

The suite uses versioned contracts for all shared projections and notification
envelopes.

The current authoritative flow is:

```text
AJRM Marine Traffic -> standard Signal K notifications -> AJRM Marine Notifications -> Audio / Display / Alerts / Console
AJRM Marine Traffic -> target/profile/audio-policy projections -> Display / Alerts / third-party Signal K clients
```

The runtime compatibility `mode: "traffic"` value may remain in projections for
existing consumers, but it is not a selectable runtime mode.

## Consequences

- AJRM Marine Traffic publishes authoritative collision notifications when enabled.
- Commands are owned by AJRM Marine Traffic and are available when the plugin is
  enabled and the user has write access.
- Display and Alerts consume AJRM Marine Traffic, Notifications, and Audio
  projections.
