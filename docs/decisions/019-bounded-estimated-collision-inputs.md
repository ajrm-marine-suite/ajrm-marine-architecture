# ADR-019: Continue collision assessment with bounded explicit estimates

Status: accepted  
Date: 2026-08-24

## Context

Nominal AIS reporting intervals are not safe receiver-side expiry limits. A
missed or collided transmission can leave a valid target observation older than
Traffic's existing 20/45/60-second moving-target fresh limits. Similarly,
Traffic previously stopped CPA/TCPA calculation as soon as live own position
exceeded 30 seconds even though Navigation Integrity maintained an operational
dead-reckoning estimate.

Reassurance also considered only Traffic and live-position state. It could say
“All's well” while Instrument Alerts had an active Warning or Danger, and a
consumer could not distinguish a healthy empty Notifications projection from a
broker that had stopped publishing.

## Decision

Traffic preserves one coherent accepted target-motion observation containing
position, SOG, COG, position timestamp and source evidence. The existing
AIS-class/motion ages remain the boundary between fresh and estimated data.
After that boundary, Traffic may linearly project the coherent observation to a
configurable total age, initially 180 seconds for moving or unknown targets and
420 seconds for stationary targets. The target remains visibly estimated and
publishes both fresh and estimate limits. CPA/TCPA is unavailable after the
estimate limit or whenever coherent motion is unavailable.

Navigation Integrity publishes one versioned
`ajrm-marine-operational-motion` object containing estimated position, SOG,
COG, estimate timestamp, age since trusted alignment, uncertainty radius,
source, GPS dependence and provenance. Traffic consumes only that object for
own-position continuation; it does not combine separate DR leaves or use the
independent integrity-comparison track. The initial configurable continuation
is 180 seconds beyond Traffic's live-position freshness limit.

While own position is estimated, Traffic continues collision calculation and
escalation. Alerts explicitly label the estimate and carry DR age, uncertainty
and source. Traffic does not publish “Traffic clear” or “All's well” from the
degraded position. Expiry means calculation unavailable, never evidence that a
risk has cleared.

Notifications republishes its complete active projection on a configurable
heartbeat, initially every 10 seconds. Traffic accepts the exact versioned
projection only within a configurable freshness limit, initially 30 seconds.
“All's well” is withheld when that projection is absent, invalid or stale, or
when provider `ajrm-marine-instrument-alerts` has an active `warning` or
`danger` envelope. Information-level instrument events do not suppress it.
The reassurance quiet period defaults to 15 minutes and restarts in full after
the final blocking alert resolves.

## Consequences

- AIS collision/reception gaps no longer discard a still-useful calculation at
  the nominal reporting interval.
- Every degraded calculation is distinguishable from fresh evidence and has a
  hard configurable expiry.
- Own-position fallback has one explicit producer-owned contract and cannot be
  confused with the independent integrity comparison.
- Safe-clear and reassurance claims remain more conservative than collision
  alert continuation.
- Notifications liveness becomes observable without interpreting alert text,
  path names or absence of messages.
