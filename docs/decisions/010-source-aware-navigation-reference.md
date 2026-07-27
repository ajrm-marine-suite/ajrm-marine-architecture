# ADR-010: Use a source-aware own-vessel navigation reference

Status: accepted

Date: 2026-07-27

## Context

The 14 and 16 July 2026 live voyages exposed several different concepts being
treated as though they were interchangeable:

- course over ground is the direction of motion over the ground;
- heading is the direction in which the bow points;
- magnetic and true angles need an explicit, selected variation source;
- speed through water plus heading is not a complete over-ground vector when
  current or leeway is present;
- a current estimate derived from GPS is not independent evidence with which to
  validate that GPS.

The test boat has no continuously available dedicated compass. At voyage start,
GPS COG is normally the only directional reference. A Simrad TP32 later
publishes magnetic heading while active. A future installation may have a
calibrated NMEA 2000 compass publishing magnetic heading, true heading, or both.

AJRM Marine Traffic currently consumes `navigation.headingTrue` and
`navigation.courseOverGroundTrue`, preferring heading whenever it has ever
received one. It does not retain per-field source or freshness. Before SK
Derived Data was enabled, the TP32's magnetic heading was invisible to Traffic
and all clock positions were COG-relative. After a true heading arrives, a stale
heading can remain preferred after the TP32 stops.

The later voyage also contained two live
`navigation.magneticVariation` sources:

- an obsolete Garmin GPSMAP value near -2.7 degrees;
- the SK Derived Data WMM value near -1.36 degrees.

Without source priority, derived true heading alternated between those
corrections. A path being present and non-null was therefore not sufficient
evidence that it was the intended source.

SK Derived Data's Set and Drift calculation is unsuitable as an AJRM DR input.
It assumes the through-water vector lies exactly along heading, does not model
leeway, does not qualify sources, and currently publishes an incorrect
cross-current set direction. More fundamentally, current calculated from GPS
COG/SOG cannot be used as independent evidence against the same GPS.

## Decision

Create a small server-side **AJRM Marine Navigation Reference** provider. This
is a focused navigation-data authority, not a charting webapp and not additional
Traffic encounter policy.

The provider will publish one versioned, namespaced own-vessel projection for
AJRM consumers. It will preserve standard Signal K units and include, for every
selected value:

- value;
- reference frame;
- source;
- source kind;
- source timestamp and age;
- selection method;
- quality or uncertainty;
- whether the value is GPS-dependent;
- fallback or degraded-state reason.

Traffic, GPS Integrity, Instruments, Display, Snapshot, DR Plotter, Capture, and
Voyage Viewer will consume this projection. Raw Signal K paths remain the public
interoperability layer and a bounded compatibility fallback during migration.

### Directional references

The provider will distinguish at least these values:

1. **Ground track**: coherent, fresh `navigation.courseOverGroundTrue` and
   `navigation.speedOverGround` from the selected GNSS source.
2. **True bow heading**: a fresh true heading from a compass, or a fresh
   magnetic heading converted using the selected magnetic variation.
3. **Clock reference**: true bow heading when trustworthy; otherwise a clearly
   labelled moving-COG proxy; otherwise unavailable.
4. **Track through water**: true heading plus signed leeway when leeway is
   available. It must not silently be equated with heading.

Selection order for true bow heading is:

1. explicitly preferred, fresh direct true heading from a calibrated compass;
2. explicitly preferred, fresh magnetic heading plus AJRM-selected WMM
   variation;
3. no bow heading.

Fresh COG may be used as a clock-reference proxy only while moving above a
configured threshold with hysteresis. It remains labelled `track-proxy`; it
must not be published or described as measured heading. At low speed, or when
COG is unstable, the clock reference becomes unavailable.

Source switching will use freshness, minimum dwell/hysteresis, and same-source
GNSS method quality, satellite, HDOP, integrity, and receiver-type evidence
rather than arrival order. Explicit source preference remains the operator
override. A stale compass must yield to a valid COG proxy; a
returning compass must become primary only after it has met the configured
stability rule.

Position, COG, and SOG are selected as one coherent, fresh, same-source ground
motion solution. Explicit no-fix or unsafe-integrity evidence makes a source
unusable even when it is preferred. A position retained only to evaluate the
magnetic model must not be published as though it were the current live
position.

A true-heading path associated with a GNSS receiver remains GPS-dependent
unless the operator explicitly identifies that exact source as an independent
integrated compass. Merely allowing GNSS-associated heading, or receiving a
dual-antenna GNSS heading, does not make it independent evidence for GPS
integrity checking.

### Magnetic variation

AJRM will calculate magnetic variation locally from position, date, and a
maintained World Magnetic Model implementation. The projection will identify
the model and epoch.

Direct NMEA variation remains visible as evidence but is selected only by
explicit source configuration. Values from different sources will never be
combined merely because they share `navigation.magneticVariation`.

SK Derived Data may still publish standard calculated paths for third-party
clients, but AJRM consumers do not rely on it. Signal K Source Priority may use
a path-level override that prefers `derived-data` for the standard
`navigation.magneticVariation` path. Physical GPS/AIS sources remain preferred
for direct true COG. A future direct true-heading compass is preferred over
derived true heading.

### Traffic clock positions

Traffic's existing relative-bearing subtraction is retained:

`relative bearing = target true bearing - selected true reference`

The correction is to the reference input and its lifecycle, not to CPA
velocity mathematics. CPA/TCPA continues to use COG/SOG because those are
over-ground motion vectors.

Traffic will:

- consume the resolved clock reference;
- consume the provider's coherent position/COG/SOG triplet for own-motion
  range and CPA calculations while the provider contract is present;
- reject stale references;
- publish `referenceKind`, source, age, and uncertainty with the encounter;
- identify COG fallback as track-relative;
- retain range, CPA/TCPA, and collision-risk alerts when using a track proxy,
  but suppress bow-relative encounter classifications and COLREG give-way,
  stand-on, or head-on instructions until a real heading is available;
- omit clock wording when no reliable relative reference exists;
- use an absolute phrase such as `bearing 090 degrees true` instead of
  `3 o'clock true` when only an absolute bearing is available;
- apply uncertainty-aware sector hysteresis so small compass noise does not
  chatter between adjacent clock positions.

### Current, set, drift, and leeway

Internally, current and residuals will be represented as north/east vectors.
If AJRM calculates an observed residual, the flow-to equations are:

```text
east  = SOG * sin(COG true) - STW * sin(track through water true)
north = SOG * cos(COG true) - STW * cos(track through water true)
drift = hypot(east, north)
set true = normalise(atan2(east, north))
```

`track through water true` is heading true plus signed leeway. When leeway is
unknown, the result contains unmodelled leeway and must be named
**ground-minus-water residual**, not tide or trusted current.

Residual estimation will use time-aligned inputs and stable-motion windows. It
will reject low-speed COG, turns, acceleration, stale samples, incoherent
sources, and outliers. It will publish age and uncertainty. TP32 compass
uncertainty is configurable; a 5-degree error at 5 knots alone represents
approximately 0.44 knots of transverse-vector uncertainty.

No AJRM component will consume arbitrary `environment.current.*` values without
checking a coherent same-source set/drift pair and explicit origin. Set is
always the direction **towards** which the current flows.

### Dead-reckoning independence

Operational fallback DR and integrity-comparison DR are separate products:

- **Operational DR** may remain GPS-locked while GPS is healthy. After an
  outage it may use heading/STW plus independently sourced current, or a frozen
  pre-outage GPS-derived residual, but it must label the latter GPS-dependent,
  decay its confidence, and grow uncertainty.
- **Integrity DR** must never use COG/SOG from the GNSS being tested, nor a
  current/residual derived from that GNSS. If independent heading/STW and
  independent current/leeway evidence are unavailable, integrity assurance is
  reported as unavailable or reduced rather than manufactured from the GPS
  under test.

When COG/SOG is used as one over-ground vector, current and leeway are already
present and are not added again. When heading/STW is used, current is added at
most once and leeway is either explicit or included in uncertainty.

## Required component corrections

### AJRM Marine Traffic

- Store source and timestamp for heading, COG, SOG, and position instead of one
  bare last value.
- Replace `headingTrue ?? cogTrue` with the resolved clock reference.
- Expire stale TP32 heading and fall back to a qualified moving COG proxy.
- Publish clock-reference provenance and degraded state.
- Replace absolute `N o'clock true` fallback wording with degrees true or no
  relative clock.
- Use the provider's coherent same-source position/COG/SOG triplet for
  own-motion geometry, without reconstructing a mixed raw fallback while a
  valid provider contract is present.
- Preserve COG/SOG CPA and relative-bearing geometry.
- Suppress bow-relative COLREG encounter advice when the only directional
  reference is a COG track proxy.

### AJRM Marine GPS Integrity

- Remove the direct use of magnetic heading as though it were true.
- Resolve GNSS position, SOG, COG, method quality, satellite count, and HDOP
  coherently rather than mixing default sources.
- Resolve compass and STW independently from the GNSS source.
- Separate operational and integrity motion-source policies.
- Prohibit COG/SOG and GPS-derived residuals from integrity DR.
- Accept current only as an atomic, fresh, same-source vector with origin and
  quality metadata.
- Stop selecting set and drift independently by whichever individual value is
  newest.
- Model leeway explicitly when available; otherwise increase uncertainty and
  expose that it is unmodelled.

### Instruments, Display, and Snapshot

- Consume the authoritative bow/track reference and show whether it is compass
  heading or a COG proxy.
- Use nullish/finite selection so a valid zero-radian heading does not fall
  through to COG.
- Do not label a GPS-derived residual as tide/current.
- Preserve source and freshness in snapshots and support evidence.
- Do not recreate server-owned clock or safety policy in the browser.

### DR Plotter, Capture, and Voyage Viewer

- Carry and display reference kind, source, age, uncertainty, current/residual
  origin, leeway status, and GPS-dependence.
- Distinguish operational fallback DR from genuinely independent integrity DR.
- Keep source data in replay bundles so current code can recompute derived
  projections.

### Console BITE and Simulator

- Verify current origin and independence, not merely that a retained current
  value exists.
- Add compass/source transition scenarios and prevent BITE synthetic sources
  from contaminating live Source Priority configuration.

## Verification

Add focused unit tests and compact, anonymised regression fixtures based on the
14 and 16 July voyages:

1. COG-only voyage start.
2. Magnetic TP32 heading appears and becomes the bow reference.
3. TP32 heading stops and expires back to moving COG.
4. Low-speed or unstable COG leaves clock reference unavailable.
5. Garmin and WMM variation coexist without true-heading alternation.
6. A direct true-heading compass outranks derived heading.
7. A valid zero-radian heading remains valid.
8. Angle wraparound at 359/0 degrees remains stable.
9. Heading 000, STW 2.5 knots, COG 090, SOG 5 knots produces residual set
   approximately 116.6 degrees true, not 243.4 degrees.
10. COG/SOG DR does not add current or leeway again.
11. Integrity DR refuses GPS-derived motion/current evidence.
12. Operational GPS-loss DR can use a retained residual only with visible
    GPS-dependence, decay, and uncertainty growth.
13. Leeway and compass uncertainty rotate/widen the result as expected.
14. Multiple GNSS sources and implausible spikes cannot create mixed vectors.

## Consequences

- Traffic remains focused on encounters instead of becoming a general
  navigation sensor-fusion plugin.
- GPS Integrity gains a defensible meaning for the word independent.
- Installations with different compass/GNSS combinations share one explicit
  policy and diagnostic projection.
- SK Derived Data remains useful during migration but is not the long-term
  navigation authority.
- A new backend component and versioned contract add work, but they remove
  duplicated and contradictory source-selection logic across the suite.

## Initial implementation

The first implementation is `signalk-ajrm-marine-navigation-reference` contract
version 1. Traffic and GPS Integrity consume it directly. Instruments and
Snapshot use it for heading-dependent presentation and evidence; Display and DR
Plotter receive the corrected Traffic/GPS Integrity projections.

The provider uses WMM 2025 from the maintained `geomagnetism` package. It
publishes instantaneous ground-minus-water residual only as GPS-dependent
diagnostic evidence. It does not promote that residual to trusted current;
stable-window estimation and any optional frozen operational residual remain
future refinements.
