# ADR-004: Signal K is the public integration architecture

Status: proposed

## Decision

The suite uses standard Signal K data paths, base units, metadata,
notifications, deltas, subscriptions, resources, request/response conventions,
security, and discovery wherever they apply.

Suite-specific envelopes and APIs are optional, namespaced, versioned, and
documented extensions. They enrich standard behavior but do not replace it.

## Consequences

- Third-party applications can consume meaningful notifications without
  installing or understanding AJRM Marine Notifications.
- Live clients use normal Signal K subscriptions where possible.
- Durable shared marine objects use Signal K resources where appropriate.
- Plugin APIs carry OpenAPI descriptions.
- A proposed extension that becomes broadly useful should be considered for
  upstream Signal K standardization.
- Private code imports and undocumented inter-plugin subtrees are not accepted
  integration contracts.
