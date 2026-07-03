# AJRM Marine Suite Compatibility Contracts

Status: cutover contract baseline
Date: 2026-06-20

These contracts let the suite evolve additively after the Traffic cutover.
A component may add fields, but it must not remove or reinterpret fields within
a contract major version.

## 1. Shared identity and ordering rules

Every provider and projection publisher creates a new opaque `sessionId` at
process start. Runtime state from an earlier session is not restored.

Every event or projection stream has a monotonic integer `sequence`:

- the first published item in a session has sequence `1`;
- sequence increases by exactly one for every accepted state change;
- repeated publication of the same snapshot does not advance sequence;
- clients compare sequence only within the same `sessionId`;
- a changed `sessionId` makes every sequence and cached runtime projection from
  the previous session stale.

The domain provider creates one opaque `correlationId` when a meaningful
condition or one-shot event is first evaluated. The identifier is copied
unchanged through the standard notification extension, AJRM Marine Notifications
projection, audio request, Audio timeline, client acknowledgement, and debug
records.

Related identifiers have distinct jobs:

| Identifier | Created by | Meaning |
| --- | --- | --- |
| `sessionId` | Each runtime component | Process/session boundary |
| `sequence` | Each projection publisher | Ordering within one stream/session |
| `correlationId` | Domain provider | End-to-end causal chain |
| `eventId` | Domain provider | Immutable provider event/update identity |
| `subjectKey` | Domain provider | Stable continuing condition identity |
| `requestId` | AJRM Marine Notifications | One audio-delivery request |
| `playbackId` | AJRM Marine Audio | One scheduled playback attempt |

Identifiers are opaque strings. Consumers must not parse meaning from them.

## 2. Standard Signal K notification contract

Providers publish under:

```text
vessels.self.notifications.<well-known-or-source-mirrored-path>
```

The standard notification remains independently useful:

```json
{
  "state": "alarm",
  "method": ["visual", "sound"],
  "message": "Collision alarm with Ferry Alpha"
}
```

Rules:

- `state` is one of `normal`, `alert`, `warn`, `alarm`, or `emergency`;
- `method` contains only standard methods such as `visual` and `sound`;
- `message` is meaningful without suite extensions;
- publishing `null` at the same path clears the active condition;
- providers use a well-known notification path or mirror the monitored source
  path where practical;
- consumers never infer provider-specific meaning from prose, path shape,
  category, or MMSI format.

The optional additive envelope remains at:

```text
notificationValue.data.ajrmMarineNotifications
```

Envelope contract major version `1` retains the existing fields and adds the
following optional fields:

```json
{
  "schemaVersion": 1,
  "provider": "ajrm-marine-traffic",
  "providerSessionId": "opaque-provider-session",
  "sourceSequence": 27,
  "correlationId": "opaque-correlation",
  "subjectKey": "ajrm-marine:vessel:235900004",
  "eventId": "opaque-provider-event",
  "revision": 27,
  "lifecycle": "active",
  "timestamp": "2026-06-19T15:30:00.000Z",
  "priority": { "level": "danger", "score": 800 },
  "supersedes": [],
  "history": { "policy": "on-resolve" },
  "delivery": {
    "visual": true,
    "audio": true,
    "preempt": true,
    "repeatSeconds": 60,
    "expiresSeconds": 90
  },
  "presentation": {
    "title": "Ferry Alpha",
    "label": "Collision alarm",
    "message": "Collision alarm with Ferry Alpha",
    "category": "collision",
    "facts": []
  },
  "actions": [],
  "context": { "mmsi": "235900004" }
}
```

Compatibility requirements:

- existing envelope version `1` consumers ignore unknown additive fields;
- `providerSessionId`, `sourceSequence`, and `correlationId` become required for
  suite providers after their contract-hardening release, but AJRM Marine
  Notifications continues accepting standard notifications and older version `1`
  envelopes without them;
- AJRM Marine Notifications supplies a broker-local correlation ID only for plain
  standard Signal K input that has none and marks its origin as
  `correlationOrigin: "broker"`;
- a provider never reuses an `eventId` for changed content;
- `revision` and `sourceSequence` are monotonic within `providerSessionId`;
- an active condition is resolved by a `resolved` envelope or standard `null`
  clear at the same path.

## 3. AJRM Marine Notifications broker projection

Canonical Signal K path:

```text
vessels.self.plugins.ajrmMarineNotifications
```

Contract:

```json
{
  "contract": "notifications-plus-projection",
  "contractVersion": 1,
  "sessionId": "opaque-broker-session",
  "sequence": 184,
  "serverTime": "2026-06-19T15:30:00.000Z",
  "active": [],
  "recentActivity": [],
  "audioRequest": {
    "sequence": 42,
    "requestId": "opaque-audio-request",
    "correlationId": "opaque-correlation",
    "subjectKey": "ajrm-marine:vessel:235900004",
    "eventId": "opaque-provider-event",
    "message": "Collision alarm with Ferry Alpha",
    "priorityScore": 800,
    "preempt": true,
    "expiresAt": "2026-06-19T15:31:30.000Z",
    "outputs": {
      "localSpeaker": true,
      "alerts": true,
      "stream": true
    }
  }
}
```

For version `1` consumers, the broker also publishes compatibility aliases:

- `history` aliases `recentActivity`;
- `lastAudioEvent` carries the provider envelope used to make `audioRequest`;
- `audioSequence` aliases `audioRequest.sequence`;
- `updatedAt` aliases `serverTime`.

Rules:

- `active` contains only currently active visual notifications;
- `recentActivity` contains resolved and one-shot in-session activity;
- active entries are not duplicated in `recentActivity`;
- broker `sequence` advances for any accepted active/activity/audio change;
- audio request sequence advances only for a newly accepted audio request;
- a broker restart creates a new session and empty runtime projection;
- HTTP status is bootstrap/recovery; Signal K subscription is the primary live
  transport.

## 4. AJRM Marine Traffic target projection

Canonical Signal K path:

```text
vessels.self.plugins.ajrmMarineTraffic.targets
```

Capability and health path:

```text
vessels.self.plugins.ajrmMarineTraffic.capabilities
```

Target projection contract:

```json
{
  "contract": "ajrm-marine-traffic-targets",
  "contractVersion": 1,
  "sessionId": "opaque-traffic-session",
  "sequence": 96,
  "generatedAt": "2026-06-19T15:30:00.000Z",
  "mode": "traffic",
  "authoritative": true,
  "source": {
    "capturePlusPlayback": false,
    "ownVesselPositionFresh": true
  },
  "targets": [
    {
      "id": "urn:mrn:imo:mmsi:235900004",
      "mmsi": "235900004",
      "name": "Ferry Alpha",
      "position": { "latitude": 50.1, "longitude": -1.2 },
      "navigation": {
        "sog": 6.2,
        "cogTrue": 1.4,
        "headingTrue": 1.39
      },
      "dimensions": {
        "length": 120,
        "beam": 22,
        "reference": "reported"
      },
      "encounter": {
        "range": 850,
        "bearingTrue": 0.8,
        "relativeBearing": 0.2,
        "cpa": 75,
        "tcpa": 210,
        "state": "alarm",
        "uiOrder": 1,
        "silenced": false,
        "correlationId": "opaque-correlation"
      },
      "freshness": {
        "updatedAt": "2026-06-19T15:29:59.500Z",
        "ageMs": 500,
        "stale": false
      }
    }
  ]
}
```

Rules:

- standard Signal K units are used: metres, seconds, metres per second, and
  radians unless a field explicitly states otherwise;
- missing values are omitted or `null`; sentinel numbers are forbidden;
- targets are keyed by stable Signal K vessel identity, with MMSI carried as
  data rather than assumed from path parsing;
- Traffic owns encounter state, ordering, silence state, and correlation;
- displays may calculate screen geometry but must not recalculate safety state;
- `mode: "traffic"` is retained as a compatibility value for the authoritative
  Traffic runtime;
- Traffic publishes standard collision notifications when enabled and the
  user has not muted/silenced the relevant target.

## 5. AJRM Marine Audio playback timeline

Canonical Signal K path:

```text
vessels.self.plugins.ajrmMarineAudio.timeline
```

Each published event is the latest event in a session-scoped ordered stream:

```json
{
  "contract": "ajrm-marine-audio-timeline",
  "contractVersion": 1,
  "sessionId": "opaque-audio-session",
  "sequence": 73,
  "event": {
    "state": "speaker-started",
    "playbackId": "opaque-playback-attempt",
    "requestId": "opaque-audio-request",
    "correlationId": "opaque-correlation",
    "subjectKey": "ajrm-marine:vessel:235900004",
    "priorityScore": 800,
    "message": "Collision alarm with Ferry Alpha",
    "assetUrl": "/signalk/v1/api/ajrmMarineAudio/audio/example.mp3",
    "scheduledStartAt": "2026-06-19T15:30:05.500Z",
    "occurredAt": "2026-06-19T15:30:05.507Z",
    "durationMs": 7400,
    "preemptedPlaybackId": null
  }
}
```

Allowed lifecycle states:

```text
accepted
queued
synthesis-started
audio-ready
playback-scheduled
speaker-started
speaker-finished
interrupted
resumed
cancelled
expired
failed
```

Rules:

- only Audio publishes claims that playback started or finished;
- every lifecycle transition advances timeline `sequence`;
- all transitions retain `requestId`, `playbackId`, and `correlationId`;
- the rendered asset referenced by `audio-ready` and `playback-scheduled` is the
  same semantic audio used by the Signal K server speaker;
- Alerts first observes the timeline without changing playback behavior;
- after measurement and promotion, Alerts follows `playback-scheduled` and
  does not maintain an independent semantic queue;
- client acknowledgements are diagnostics, not authority over Signal K server playback;
- a missing or late Alerts must not delay the local speaker beyond the
  configured bounded synchronization window.

## 6. Debug-log contract

Consequential transitions use Signal K `app.debug()` or `app.error()` and one
single-line key/value shape:

```text
event=broker.accept sessionId=... sequence=184 correlationId=... eventId=... subjectKey=... elapsedMs=7
```

The required keys are the identifiers available at that stage plus `event`.
Logs must not contain credentials, invitation codes, full private
configuration, or high-frequency per-refresh noise.

AJRM Marine Capture replay evidence must include source vessel paths, standard
notifications, broker projection, Traffic projection, and Audio timeline paths.

## 7. Version negotiation and failure behavior

- `contractVersion` is a major integer.
- New optional fields are additive within a major version.
- Removing fields, changing units, changing lifecycle meaning, or changing
  ordering semantics requires a new major version and a parallel compatibility
  period.
- Consumers ignore unknown fields and reject unsupported major versions.
- When an enhanced projection is unavailable or unsupported, consumers fall
  back to the documented standard Signal K surface without silently
  reconstructing provider-specific policy.
