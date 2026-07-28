# ADR-011: Replay physical sensor sources and capture recomputed child voyages

Status: accepted; operational-recovery amendment released

Date: 2026-07-27

Updated: 2026-07-28

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

### 28 July replay incident

During a real-time recomputation attempt, replay stopped advancing immediately
after injection of a historical `navigation.datetime` value. That exact
sequence is direct evidence. The live
`~/.signalk/plugin-config-data/set-system-time.json` configuration confirms
that Set System Time was enabled with `configuration.interval: 0`,
`configuration.sudo: true`, and
`configuration.preferNetworkTime: true`. It was therefore armed to act on the
first `navigation.datetime` received after startup and had permission to adjust
the host clock.

`preferNetworkTime: true` skips that adjustment only when chrony reports a
valid time source; the option alone does not prevent a replayed time value from
being applied. Given the enabled configuration, the exact stall immediately
after the historical datetime, and Logger's then wall-clock-based scheduler,
the incident is diagnosed: Set System Time moved the host clock backwards and
Logger's pacing calculation waited against the resulting negative elapsed
wall time.

The incident also left the operator with Capture apparently recording while
Logger reported no recording loaded, and with stop/build controls disabled.
That state must be recoverable without labelling a partial run as a verified
recomputation.

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

`navigation.datetime` is a data value rather than an update timestamp, so it
must be rewritten explicitly to replay wall time. A replay must never publish a
historical system-time input that can move the host clock. The parent recording
and Logger's explicit original-time projection retain the historical value for
audit and WMM/voyage analysis.

Playback rate and delay calculations use a monotonic elapsed-time source.
Wall-clock time remains the source of fresh outgoing Signal K timestamps, but a
host-clock correction cannot make the playback scheduler wait for hours, run
backwards, or send a compensating burst.

Voyage replay retains the configured warm-up interval before the recorded voyage
start so source selection, static AIS data, and plugin state can settle.
One-times speed is enforced while a recomputed child capture is active so
timer-, freshness-, hysteresis-, and human-facing behavior retain their intended
timing. Accelerated replay remains useful for structural debugging outside that
capture workflow, but is not timing-equivalent.

## Recomputed child voyage

Capture can start a dedicated recomputed-replay recording after a sensor-only
voyage has been loaded at its prepared start. Starting it is one coordinated
action: Capture fixes playback at `1x`, starts the result recording, and
commands Logger to play. The operator does not need a separate Logger Play
action. This recording:

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

Capture shows the active phase and Logger cursor/total progress. A disabled
stop/build control must have a visible reason, such as playback still running,
calculation flush pending, or no recomputed voyage active; a grey control alone
is not an adequate state report.

The child `index.json` records the parent voyage, playback mode and rate,
original time range, source rule and resolved source IDs, filter statistics,
source catalogue, cursor/completeness coverage, software snapshots, and
live-input-isolation result. Capture does not finalise a normal recomputed ZIP
until Logger reaches the end of the loaded voyage; an aborted ordinary stop is
therefore distinguishable by `coverage.complete: false`.
Coverage is cumulative across all prepared segments, so completing the first
hourly file cannot be mistaken for completing a multi-segment voyage.

An explicit Cancel action stops playback and result capture and still packages
the evidence collected so far. The resulting ZIP records the cancellation
reason, partial coverage, and an explicit incomplete/unverified result. It must
not be accepted by comparison tooling as a successful recomputation.

If Signal K or the Pi is interrupted, Capture startup recovery preserves a
bounded amount of partial recording, status, event, and diagnostic evidence.
Recovery must close or quarantine unfinished files safely, retain the parent
lineage and last known coverage when available, and mark the recovered bundle
incomplete/unverified. It must not copy an unbounded system journal or silently
discard all evidence merely because normal finalisation did not run.

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

## Package-operation guard

The Boat Bootstrap installer and updater perform a read-only local Signal K
preflight before changing plugin packages or restarting Signal K. They refuse
the operation while Logger is recording, playback is active or paused, a
replay-result capture is active, or Capture has an active voyage. A reachable
server whose state cannot be verified fails closed. Only a genuinely
unreachable server with no active Signal K service may fail open with a
warning. An explicit force override exists for deliberate recovery, with a
warning that in-memory replay and voyage work may be lost.

## Verification

1. Disable/disconnect live inputs, restart Signal K, and keep the calculation
   plugins under test enabled.
2. Load the 14 July voyage and confirm long `YDEN.<device-id>` sources resolve
   into the exact allow-list.
3. Load the 16 July voyage and confirm `derived-data`, Traffic, GPS Integrity,
   Audio, route/course providers, notifications, and prior `plugins.*` values
   are excluded.
4. Confirm source labels are unchanged, update timestamps are fresh, and
   replayed `navigation.datetime` follows replay wall time rather than the
   parent voyage date.
5. Confirm the replay clock retains each original recorded timestamp.
6. Start recomputed capture in Capture and confirm it restarts and begins Logger
   playback automatically at `1x`.
7. Confirm progress and the stop/build disabled reason remain visible through
   playback and the calculation flush.
8. Confirm monotonic pacing remains stable across a controlled host wall-clock
   adjustment.
9. Stop after the end-of-capture calculation flush and inspect the child ZIP.
10. Confirm the child contains filtered sensor inputs, new Navigation Reference,
   Traffic, GPS Integrity/DR, and other calculated outputs, but no old calculated
   inputs.
11. Confirm `coverage.complete` is true.
12. Cancel a separate test run and confirm its ZIP is preserved but explicitly
    incomplete/unverified.
13. Interrupt a separate test run and confirm startup recovery preserves only
    bounded partial evidence and never marks it verified.
14. Confirm both package scripts refuse an update during each protected
    Logger/Capture state and require the explicit override to proceed.
15. Reject validation runs whose isolation metadata reports physical live input.

## Consequences

- A voyage can test the current code rather than replaying its historical
  answers.
- Source policy is explicit and auditable across boats with different hardware.
- Original and wall-clock time remain available for their separate purposes.
- Historical navigation time cannot be fed to a host-clock setter, and pacing
  is independent of host wall-clock corrections.
- Recomputed ZIPs form a traceable parent/child chain that can be compared
  without expecting byte-for-byte equality.
- Cancelled and interrupted runs retain useful bounded evidence without being
  confused with complete validation.
- Routine package updates cannot silently destroy active replay/capture state.
- Reliable validation requires an operational step outside the software:
  isolating the Pi from live navigation inputs for the duration of replay.
