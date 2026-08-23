# AJRM Marine Suite Compatibility Contracts

Status: cutover contract baseline
Date: 2026-08-23

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

### Anchoring depth lifecycle distinction

Anchoring depth callouts and **Anchor dropped** announcements are one-shot
events. They publish at:

```text
vessels.self.notifications.environment.depth.callout
```

Their AJRM Marine Notifications envelope uses `lifecycle: "event"`, a new
`eventId` for each announcement, and the stable subject
`ajrm-marine-instrument-alerts:anchoring-depth-callout`. A newer event for that
subject replaces an older queued callout. The provider publishes a standard
`null` clear 30 seconds after the latest event, resetting the timer when a
newer event arrives, or clears sooner when the measured depth rises above the
configured callout band plus hysteresis.

This lifecycle is independent of the continuing shallow-depth monitor at:

```text
vessels.self.notifications.environment.depth.belowKeel
```

That notification uses the normal active/resolved threshold lifecycle and
remains active until its own threshold and hysteresis clear. Clearing a
one-shot callout must never clear or conceal the continuing shallow-depth
condition.

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
      "aisClass": "A",
      "aisClassEvidence": {
        "status": "reported",
        "inputPath": "sensors.ais.class",
        "source": "YDEN.43",
        "timestamp": "2026-06-19T15:29:59.500Z",
        "provenance": {
          "$source": "YDEN.43",
          "source": {
            "label": "YDEN",
            "type": "NMEA2000",
            "pgn": 129038,
            "src": "43",
            "sentence": null,
            "talker": null
          }
        },
        "observations": [
          {
            "value": "A",
            "source": "YDEN.43",
            "timestamp": "2026-06-19T15:29:59.500Z",
            "trusted": true,
            "qualification": "physical-nmea2000-ais",
            "provenance": {
              "$source": "YDEN.43",
              "source": {
                "label": "YDEN",
                "type": "NMEA2000",
                "pgn": 129038,
                "src": "43",
                "sentence": null,
                "talker": null
              }
            }
          }
        ],
        "untrustedObservations": []
      },
      "position": { "latitude": 50.1, "longitude": -1.2 },
      "navigation": {
        "sog": 6.2,
        "cogTrue": 1.4,
        "headingTrue": 1.39,
        "rateOfTurn": 0.004
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
- `aisClass` is exactly `"A"`, `"B"`, or `null`; it is never inferred from
  MMSI, dimensions, ship type, rate of turn, or update frequency;
- `aisClassEvidence.status` is `reported`, `unknown`, or `conflicting`;
  conflicting fresh trusted Class A and Class B reports produce
  `aisClass: null`;
- only direct physical AIS receiver reports whose structured provenance
  identifies an appropriate NMEA 2000 AIS class-message PGN or an NMEA 0183
  VDM/VDO sentence can determine `aisClass`; unsupported attributed reports
  remain visible under `untrustedObservations`, while unattributed and known
  Vessel Database copies do not become independent evidence;
- `navigation.rateOfTurn` is radians per second. An explicit projected `null`
  is a clear/tombstone: clients must remove any cached turn indication rather
  than retaining the last numeric rate. Absence of the additive field may
  indicate an older compatible publisher;
- Traffic owns encounter state, ordering, silence state, and correlation;
- displays may calculate screen geometry but must not recalculate safety state;
- `mode: "traffic"` is retained as a compatibility value for the authoritative
  Traffic runtime;
- Traffic publishes standard collision notifications when enabled and the
  user has not muted/silenced the relevant target.

## 5. AJRM Marine Capture voyage observations

The active voyage observation log is newline-delimited JSON at:

```text
observations/observations.jsonl
```

Each line is independently parseable observation schema version `1`:

```json
{
  "schemaVersion": 1,
  "id": "observation-20260619T153005Z-a1b2c3d4",
  "voyageId": "voyage-20260619T150000Z",
  "recordedAt": "2026-06-19T15:30:05.000Z",
  "voyageElapsedSeconds": 1805,
  "replayOriginalAt": "2026-07-16T09:34:12.000Z",
  "replayOriginalAtSource": "logger.playback.originalCapturedAt",
  "source": "ajrm-marine-display",
  "text": "Target turn arrow remained after a null ROT report.",
  "evidence": {
    "requested": true,
    "captured": true,
    "fileName": "observations/evidence/observation-20260619T153005Z-a1b2c3d4.json",
    "snapshotPreset": "debug"
  },
  "evidenceError": null
}
```

Rules:

- the bounded text note, `recordedAt`, and voyage elapsed time are the durable
  record; during replay, `replayOriginalAt` preserves Logger's explicit
  original cursor time separately from wall time;
- optional evidence is a structured AJRM Marine Snapshot using the `debug`
  preset, not a screen image, and is stored below
  `observations/evidence/`;
- Snapshot failure must not discard the text note; Capture records
  `evidence.captured: false` and a bounded `evidenceError`;
- appending the JSONL line is the commit point. A later index or live-status
  refresh failure is a post-commit warning and must not invite an automatic
  retry that could duplicate the note;
- `index.json` summarises child observation count, evidence count, errors, and
  time range; portable ZIP rebuilding preserves the log and evidence files.

For a recomputed child voyage, observations made during the replay belong only
to the child log above. If the parent ZIP contains observations, Capture writes
an additional lineage-only copy at:

```text
observations/parent-observations.jsonl
```

Parent lineage records are not counted as child observations. Their child-side
`evidence.captured` is `false` and `evidence.fileName` is `null`. A
`lineage` object names the parent voyage and records whether Capture verified a
safe referenced evidence entry in that parent. The optional
`lineage.parentEvidenceFileName` refers only to a verified entry in the named
parent ZIP; parent evidence is not copied into the child, and an unsafe or
missing reference must never become a dangling child evidence path.

## 6. AJRM Marine Audio playback timeline

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

## 7. Debug-log contract

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

## 8. Version negotiation and failure behavior

- `contractVersion` is a major integer.
- New optional fields are additive within a major version.
- Removing fields, changing units, changing lifecycle meaning, or changing
  ordering semantics requires a new major version and a parallel compatibility
  period.
- Consumers ignore unknown fields and reject unsupported major versions.
- When an enhanced projection is unavailable or unsupported, consumers fall
  back to the documented standard Signal K surface without silently
  reconstructing provider-specific policy.

## 9. Shared planning and environment contracts

The current Tidal Database in-process, status and diagnostic surfaces are:

```text
ajrm-marine-tidal-database-service-v2
ajrm-marine-tidal-database-status-v2
ajrm-marine-tidal-database-diagnostics-v2
```

Version 2 is the gate-free boundary. It owns provider/station data, port
prediction definitions, secondary corrections, region relationships, cache and
tide resolution. The removed gate methods are not an additive v1 change.

Tidal durable definitions version 3 replaces the old ambiguous duplicated
`name` with `cachedLocationName`. Service projections obtain `name` from the
exact Location UUID when available and expose `nameSource: "location"`; only a
temporary unavailable join may expose the stored fallback with
`nameSource: "cached"`. Definition writes validate the required Location type
and do not accept an independently editable display name. A port Location has
exactly one of `tidalStandardPort` or `tidalSecondaryPort`; a correction port is
operational only when its complete parent chain remains joined and terminates
at a provider-backed standard port.

While retired gate rows await verified Planning acknowledgement, Tidal
Database keeps the quarantine file genuinely readable as the legacy v1
contract, including legacy `name` fields. Only acknowledgement rewrites it as
the gate-free v3 durable contract.

Planning's Location-mutation guard protects both exact UUID joins in every live
flat row: the gate remains `tidalGate` and its reference port remains
`tidalStandardPort`. A consumer that does not understand the guard's required
reference-port protection must not claim that a catalogue mutation is safe.

The Planning tide endpoint verifies the resolver contract and exact selected
port before returning events. Its response repeats the exact reference-port UUID
and name and includes the associated reference levels. Consumers compare these
explicit fields; they do not infer the port from station name, event content or
the gate position.

Display distinguishes `fresh` and `last-known` from explicit own-position age
against a 30-second limit; missing age evidence fails closed as `last-known`.
Weather Database identifies whether the selected primary forecast is network,
exact-cache fallback or nearest-cache fallback. A contributing secondary cache
does not change the primary forecast's resolution mode.
