# July 2026 voyage software baseline

Status: recorded evidence

Date recorded: 2026-07-27

## Evidence

This baseline comes from the `installedApps` object in each voyage's start
snapshot, with Capture and Logger versions independently confirmed by
`system/start-status.json`:

- `voyage-20260714T143319Z.zip`:
  `snapshots/20260714T143319Z-start.json`
- `voyage-20260716T090451Z.zip`:
  `snapshots/20260716T090452Z-start.json`

The ZIP files were rebuilt for transfer on 26 July 2026, but the embedded
snapshots preserve the software installed when the voyages were recorded on 14
and 16 July.

## Installed versions

All 18 AJRM Marine packages listed below were version `0.6.0` in both voyages.
Their recorded source specification was
`github:ajrm-marine-suite/<repository>#v0.6.0`.

| Package | 14 July | 16 July |
| --- | --- | --- |
| `signalk-ajrm-marine-alerts` | `0.6.0` | `0.6.0` |
| `signalk-ajrm-marine-audio` | `0.6.0` | `0.6.0` |
| `signalk-ajrm-marine-capture` | `0.6.0` | `0.6.0` |
| `signalk-ajrm-marine-console` | `0.6.0` | `0.6.0` |
| `signalk-ajrm-marine-display` | `0.6.0` | `0.6.0` |
| `signalk-ajrm-marine-dr-plotter` | `0.6.0` | `0.6.0` |
| `signalk-ajrm-marine-gps-integrity` | `0.6.0` | `0.6.0` |
| `signalk-ajrm-marine-harbour-editor` | `0.6.0` | `0.6.0` |
| `signalk-ajrm-marine-instrument-alerts` | `0.6.0` | `0.6.0` |
| `signalk-ajrm-marine-instruments` | `0.6.0` | `0.6.0` |
| `signalk-ajrm-marine-logger` | `0.6.0` | `0.6.0` |
| `signalk-ajrm-marine-notifications` | `0.6.0` | `0.6.0` |
| `signalk-ajrm-marine-pi-controller` | `0.6.0` | `0.6.0` |
| `signalk-ajrm-marine-simulator` | `0.6.0` | `0.6.0` |
| `signalk-ajrm-marine-snapshot` | `0.6.0` | `0.6.0` |
| `signalk-ajrm-marine-traffic` | `0.6.0` | `0.6.0` |
| `signalk-ajrm-marine-vessel-database` | `0.6.0` | `0.6.0` |
| `signalk-ajrm-marine-voyage-viewer` | `0.6.0` | `0.6.0` |

External Signal K packages recorded in the same snapshots were:

| Package | 14 July | 16 July | Recorded source |
| --- | --- | --- | --- |
| `@signalk/resources-provider` | `1.5.1` | `1.5.1` | `^1.5.1` |
| `signalk-charts-provider-simple` | `2.4.0` | `2.4.0` | `^2.4.0` |
| `signalk-derived-data` | not installed | `1.45.0` | `^1.45.0` |

`signalk-ajrm-marine-navigation-reference` did not yet exist and therefore was
not installed in either voyage.

The 16 July recording contains deltas whose source is `derived-data`, confirming
that SK Derived Data was active during that voyage. The snapshot records the
installed version but not the plugin's individual calculation checkboxes.

The snapshots do not record the Signal K Server package version, so no server
version is inferred.

## Interpretation

The heading, clock-position, source-mixing, and DR errors found in these voyages
are therefore regressions against the coordinated AJRM Marine `v0.6.0`
baseline. Recomputed child voyages must retain this parent baseline and record
the newly installed package versions so comparisons do not confuse software
changes with sensor-input changes.

## Correction release

The source-aware navigation and sensor-only replay corrections are released as:

| Package | Correction release |
| --- | --- |
| `signalk-ajrm-marine-navigation-reference` | `v0.1.0` (new) |
| `signalk-ajrm-marine-traffic` | `v0.6.1` |
| `signalk-ajrm-marine-display` | `v0.6.1` |
| `signalk-ajrm-marine-gps-integrity` | `v0.6.1` |
| `signalk-ajrm-marine-dr-plotter` | `v0.6.1` |
| `signalk-ajrm-marine-snapshot` | `v0.6.1` |
| `signalk-ajrm-marine-instruments` | `v0.6.1` |
| `signalk-ajrm-marine-logger` | `v0.6.1` |
| `signalk-ajrm-marine-capture` | `v0.6.1` |
| `signalk-ajrm-marine-voyage-viewer` | `v0.6.1` |
| `signalk-ajrm-marine-console` | `v0.6.1` |
| private `signalk-boat-bootstrap` | `v2.0.1` |

The other AJRM packages receive no new tag from this correction. The updater
still resolves the newest existing tag in each repository; notably,
`signalk-ajrm-marine-audio` already had the unrelated `v0.6.2` release even
though both recorded voyages used `v0.6.0`. SK Derived Data remains `1.45.0`,
the current npm release on 27 July 2026, and is installed for standard-path
compatibility rather than as the AJRM navigation authority.

## Field-test follow-up release

The additional findings from review of the same voyages and subsequent live
testing are addressed by this additive follow-up:

| Package | Follow-up release | Correction |
| --- | --- | --- |
| `signalk-ajrm-marine-display` | `v0.6.2` | Render explicit Class A/B/unknown targets, clear a null ROT indicator, allow own-vessel icon sizing, and record voyage observations |
| `signalk-ajrm-marine-traffic` | `v0.6.2` | Accept AIS class only from qualified receiver provenance and preserve explicit null rate of turn |
| `signalk-ajrm-marine-gps-integrity` | `v0.6.2` | Publish Signal K units metadata for trusted SOG, COG, heading, and DR fields |
| `signalk-ajrm-marine-capture` | `v0.6.2` | Store timestamped observation JSONL, optional structured Snapshot evidence, and recomputed parent lineage |
| `signalk-ajrm-marine-instrument-alerts` | `v0.6.1` | Treat anchoring depth callouts as expiring one-shot events without clearing the continuing below-keel alert |

These tags do not change the historical `v0.6.0` software attribution for the
14 and 16 July source voyages. A recomputed child voyage records the newer
installed versions in its own snapshots and keeps the parent voyage reference
for comparison.

## Operational-recovery release

Status: released on 28 July 2026

The replay-clock and interrupted-work recovery identified during the first
real-time recomputation test are addressed by this small coordinated release:

| Package | Release | Recovery scope |
| --- | --- | --- |
| `signalk-ajrm-marine-logger` | `v0.6.2` | Rewrite replayed `navigation.datetime` to replay wall time, pace from monotonic elapsed time, name/cache corrupt-file metadata failures, validate gzip before replacing plain data, and expose explicit playback failure/abort state |
| `signalk-ajrm-marine-capture` | `v0.6.3` | Auto-start prepared recomputation at `1x`, show progress and disabled reasons, cancel into an incomplete/unverified ZIP, and preserve bounded partial startup-recovery evidence |
| private `signalk-boat-bootstrap` | `v2.0.2` | Refuse package mutation/restart while Logger or Capture has protected active work, distinguish a clean first install from unreadable installed-plugin state, and provide an explicit recovery override |

These tags do not prove that the Pi has been updated. Installation and incident
reports must continue to cite the versions actually returned by the Pi and
captured in each voyage.
