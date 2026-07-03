# AJRM Marine Implementation and Verification Plan

## Current Objective

Prepare the AJRM Marine public repositories for wider Signal K testing while
keeping each plugin independently installable and diagnosable.

Console is the suite entry point. BITE, Capture, Logger, Simulator, and Voyage
Viewer form the repeatable test and diagnostic loop.

## Release Gates

Before tagging or publishing a package:

1. The local working tree is clean except for the intended change.
2. `npm test` passes.
3. `npm pack --dry-run` includes runtime assets and excludes private material.
4. The package metadata includes Signal K keywords, display metadata, repository
   links, author, licence, and files.
5. The GitHub Actions Signal K plugin CI run is green.
6. A fresh Signal K install can activate the package with schema defaults.

## Suite Verification Gates

Before broader announcement:

1. Install mandatory apps on a clean Signal K server.
2. Confirm optional apps can be missing without breaking Console.
3. Run Console BITE with no live data feed.
4. Run BITE groups individually after fixing failures.
5. Start Capture before longer BITE or soak runs and stop it after the final
   audible summary.
6. Confirm Voyage Viewer can analyse the generated bundle.
7. Replay selected voyages through Logger at `1x`, `10x`, `20x`, and `Max`
   where practical.
8. Confirm Display, Audio, OpenCPN message display, and desktop Audio Player
   show/hear the same notification content within expected timing.

## Simulator Test Sequence

1. Start with simulator output off.
2. Confirm BITE preflight blocks if simulator or live data is active.
3. Run stationary harbour tests with stationary automute enabled.
4. Run moving harbour/coastal tests and confirm automute debounce.
5. Exercise AIS encounters: head-on, crossing port/starboard, overtaking,
   same-course, stationary own vessel, stationary target, close quarters, and
   advisory-only cases.
6. Exercise own-vessel modes: docked/anchored, self-steering, GPX route
   following, GPS normal/lost/intermittent/spoof/jump, heading on/off.
7. Exercise environment: depth, wind, tide/current, and instrument alert
   thresholds.
8. Confirm Capture comments, snapshots, and bundle contents.

## GPS and DR Test Sequence

1. Start with healthy GPS and steady speed.
2. Confirm GPS Integrity accepts fixes and counters remain sensible.
3. Turn GPS lost and confirm a lost-GPS event is counted once.
4. Confirm DR Plotter immediately records an estimated position at loss.
5. Confirm DR uses last reliable current set/drift after GPS loss.
6. Restore GPS and confirm an electronic fix is recorded immediately.
7. Confirm uncertainty is hidden or bounded while GPS is healthy.
8. Confirm Voyage Viewer summarises GPS Integrity events and DR uncertainty.

## Audio Test Sequence

1. Confirm Piper availability or clear degraded status.
2. Confirm Sound Check bypasses mute.
3. Confirm Traffic mute/automute controls ordinary suite announcements.
4. Confirm collision alarms are not dropped by lower-priority announcements.
5. Confirm stale queued announcements are superseded only by a newer event of
   the same subject or an explicit lifecycle rule.
6. Confirm server speaker, stream, browser fallback, and desktop player are
   sinks of the same Audio timeline.
7. Confirm the final BITE audible summary is spoken after preceding safety
   alerts unless the output itself is unavailable.

## Replay Expectations

Logger replay should publish source data rather than derived AJRM outputs where
possible. That allows current plugins to recompute warnings and diagnostics from
the captured voyage.

At `1x`, replay should closely reproduce the original timing. Faster replay is
useful for debugging but may not preserve every human-facing timing detail.

## Rollback

Rollback remains component-level:

- disable the failed plugin;
- reinstall the previous tagged version;
- keep the voyage/capture bundle that exposed the failure;
- add or update a BITE or unit test before re-releasing.
