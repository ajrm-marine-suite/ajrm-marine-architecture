# ADR-011: Replay physical sensor sources and capture recomputed child voyages

Status: accepted

Date: 2026-07-27

## Context

AJRM Marine voyage captures contain both physical sensor deltas and outputs
calculated by Signal K plugins. Replaying all of them together feeds old
Traffic, heading, DR, notification, and Derived Data results back into the new
software. That can hide a correction, duplicate an output, or make a stale
calculation win over the calculation under test.

The 14 and 16 July 2026 captures also demonstrate why path-based filtering is
insufficient. Physical GNSS and compass sources publish normal navigation paths,
but calculation plugins publish some of the same paths. Source identity, not the
path alone, distinguishes recorded sensor evidence from recorded calculations.
The older voyage uses long YDEN source IDs while the later voyage also uses
short IDs such as `YDEN.43` and `YDEN.4`.

Replay freshness and historical meaning are separate requirements. Signal K
consumers need fresh wall-clock timestamps so their stale-data logic works,
while WMM and voyage analysis need the original recorded time.

## Decision

AJRM Marine Logger provides two explicit modes:

1. **Standard replay** retains the existing diagnostic behavior.
2. **Sensor sources only (recompute)** injects only physical-source updates and
   lets the currently installed software recreate calculated outputs.

Sensor-only mode scans the recording's source catalogue before replay. It
resolves configured physical-source prefixes, `YDEN` by default, into an exact
allow-list of source IDs present in that recording. Users may add exact IDs or
replace prefixes for other hardware. Missing and unlisted sources are excluded.
Requested exact IDs that do not occur anywhere in the loaded voyage are shown
as not recorded and are not admitted to the resolved allow-list.
The selected rule, complete source catalogue, resolved exact list, and per-source
filter counts are published in Logger status and the replay clock.

`plugins.*` and `notifications.*` are always excluded from replay input, even if
their recorded source is accidentally selected. Original source identity is
preserved. Update and embedded Signal K timestamps are refreshed to wall time;
the Logger playback clock publishes `originalCapturedAt` separately.

Voyage replay retains the configured warm-up interval before the recorded voyage
start so source selection, static AIS data, and plugin state can settle.
One-times speed is enforced while a recomputed child capture is active so
timer-, freshness-, hysteresis-, and human-facing behavior retain their intended
timing. Accelerated replay remains useful for structural debugging outside that
capture workflow, but is not timing-equivalent.

## Recomputed child voyage

Capture can start a dedicated recomputed-replay recording after a sensor-only
voyage has been loaded and restarted. This recording:

- has zero rolling-buffer backfill;
- writes each filtered replay input once;
- writes newly emitted calculated/plugin outputs;
- excludes Logger's replay clock and its own injection feedback;
- pre-indexes every parent-voyage segment before the run and forces progression
  through all of them even when ordinary auto-advance is disabled;
- preserves recorded timing gaps across capture-segment boundaries;
- prevents pause, seek, restart, and speed changes while the result is active;
- waits for three seconds without newly calculated output after the final replay
  input, bounded to fifteen seconds;
- always builds a portable ZIP.

The child `index.json` records the parent voyage, playback mode and rate,
original time range, source rule and resolved source IDs, filter statistics,
source catalogue, cursor/completeness coverage, software snapshots, and
live-input-isolation result. Capture does not finalise a normal recomputed ZIP
until Logger reaches the end of the loaded voyage; an aborted ordinary stop is
therefore distinguishable by `coverage.complete: false`.
Coverage is cumulative across all prepared segments, so completing the first
hourly file cannot be mistaken for completing a multi-segment voyage.

Physical live inputs must be disabled or disconnected while replay runs.
Logger quarantines detected physical-source deltas from the child log and marks
the isolation result invalid, but it cannot prevent an external live delta from
reaching another plugin before Logger observes it. A contaminated run is
evidence of a test setup problem and must not be treated as deterministic.

For a deterministic comparison, disconnect or disable those inputs and restart
Signal K before loading the parent voyage. This clears prior live plugin state;
the recorded warm-up interval then establishes the calculation state used by
the replay. Logger cannot provide a universal reset hook for every third-party
or suite plugin.

## Verification

1. Disable/disconnect live inputs, restart Signal K, and keep the calculation
   plugins under test enabled.
2. Load the 14 July voyage and confirm long `YDEN.<device-id>` sources resolve
   into the exact allow-list.
3. Load the 16 July voyage and confirm `derived-data`, Traffic, GPS Integrity,
   Audio, route/course providers, notifications, and prior `plugins.*` values
   are excluded.
4. Confirm source labels are unchanged and replayed timestamps are fresh.
5. Confirm the replay clock retains each original recorded timestamp.
6. Start result capture only from the replay start with sensor-only mode active.
7. Replay at 1x with physical inputs isolated.
8. Stop after the end-of-capture calculation flush and inspect the child ZIP.
9. Confirm the child contains filtered sensor inputs, new Navigation Reference,
   Traffic, GPS Integrity/DR, and other calculated outputs, but no old calculated
   inputs.
10. Confirm `coverage.complete` is true.
11. Reject validation runs whose isolation metadata reports physical live input.

## Consequences

- A voyage can test the current code rather than replaying its historical
  answers.
- Source policy is explicit and auditable across boats with different hardware.
- Original and wall-clock time remain available for their separate purposes.
- Recomputed ZIPs form a traceable parent/child chain that can be compared
  without expecting byte-for-byte equality.
- Reliable validation requires an operational step outside the software:
  isolating the Pi from live navigation inputs for the duration of replay.
