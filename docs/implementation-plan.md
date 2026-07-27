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
2. Confirm position, SOG, COG, fix quality, and timestamps come from one
   coherent selected GNSS source.
3. Confirm the integrity DR input excludes COG/SOG and residual/current derived
   from the GNSS being tested.
4. Confirm GPS Integrity reports reduced or unavailable independent assurance
   when no independent heading/STW evidence exists.
5. Turn GPS lost and confirm a lost-GPS event is counted once.
6. Confirm DR Plotter immediately records an estimated position at loss.
7. Confirm operational DR uses only a qualified external current or a visibly
   GPS-dependent retained residual with age, decay, and uncertainty.
8. Confirm COG/SOG propagation does not add current or leeway a second time.
9. Confirm heading/STW propagation applies current at most once and exposes
   whether leeway is modelled.
10. Restore GPS and confirm an electronic fix is recorded immediately.
11. Confirm uncertainty is hidden or bounded while GPS is healthy.
12. Confirm Voyage Viewer summarises GPS Integrity events, DR independence,
    source changes, residual origin, and uncertainty.

## Navigation Reference Test Sequence

1. Start moving with COG/SOG but no compass and confirm the reference is
   explicitly `track-proxy`, not measured heading.
2. Introduce fresh magnetic TP32 heading and WMM variation; confirm the bow
   reference changes to converted true heading.
3. Stop TP32 heading and confirm it expires back to moving COG rather than
   remaining cached.
4. Stop or slow the vessel and confirm unreliable COG is not used as bow
   heading.
5. Publish simultaneous obsolete-device and Derived Data variation; confirm
   AJRM's local WMM result remains stable and neither calculated path becomes
   physical sensor evidence.
6. Publish direct true heading from a simulated calibrated NMEA 2000 compass;
   confirm it outranks derived heading and can fall back cleanly.
7. Exercise 359/0-degree wraparound, valid zero heading, source-switch
   hysteresis, and clock-sector boundaries.
8. Confirm no-reference encounter wording uses an absolute true bearing or
   omits clock wording.
9. Validate ground-minus-water residual direction, leeway handling, source
   coherence, steady-motion filtering, and uncertainty.

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

Use Logger **Sensor sources only (recompute)** mode for correction validation.
Confirm its recorded source catalogue resolves the configured physical-source
prefixes to the expected exact IDs. Missing/unlisted sources, `plugins.*`, and
`notifications.*` must be excluded.

Replay fixtures should preserve source identifiers and timestamps so navigation
reference arbitration and DR independence can be tested. Compact anonymised
extracts from the 14 and 16 July 2026 voyages should cover COG-only startup,
TP32 arrival/loss, and competing magnetic-variation sources.

At `1x`, replay should closely reproduce the original timing. Faster replay is
useful for debugging but may not preserve every human-facing timing detail.

Before a result run, disable or disconnect live sensor inputs, restart playback,
then start **Recomputed voyage replay** in Capture. At the end, let Capture stop
the Logger result recording after its calculation flush and build a portable
child ZIP. Reject a run if its live-input-isolation metadata reports physical
updates. Compare source transitions, reference kinds, clock sectors, DR
provenance, and alert decisions rather than expecting byte-identical files.

## Rollback

Rollback remains component-level:

- disable the failed plugin;
- reinstall the previous tagged version;
- keep the voyage/capture bundle that exposed the failure;
- add or update a BITE or unit test before re-releasing.
