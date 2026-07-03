# ADR-008: Replace control-heavy alerts clients with a read-only Alerts viewer

Status: accepted  
Date: 2026-06-21

## Decision

Create `signalk-ajrm-marine-alerts` as the crew-safe small-screen alert viewer.
Older companion/watch-style clients are historical fallbacks and should not
receive new operational controls.

AJRM Marine Alerts is read-only from the boat's point of view:

- it consumes AJRM Marine Notifications active and recent projections;
- it may show read-only Traffic/Audio status when useful;
- it stores only local device preferences such as layout mode, font size and
  local browser/headphone audio enablement;
- it does not expose profile selection, profile sensitivity, silence/unsilence,
  automute, Traffic settings, Notifications history clearing, or Audio
  output configuration.

The only interactive sound control in AJRM Marine Alerts is local to that
browser/device. Enabling sound on an iPhone, iPad or watch-sized browser arms
that device for local notification speech/headphone playback; it does not
change vessel mute state, Signal K server speaker output, Audio stream
settings, or Traffic policy.

## Rationale

Operational settings had become distributed across Display, Audio, Traffic, and
small-screen clients. That made the suite harder to explain, harder to secure,
and easier for crew devices to accidentally change vessel alerting behaviour.

Centralising operational policy in Traffic and Audio gives a cleaner
authority model:

- Traffic owns profiles, sensitivity, auto profile, silence state,
  stationary automute and alert policy.
- Audio owns boat audio outputs, Piper, stream and speaker configuration.
- Display owns chart/view preferences.
- AJRM Marine Alerts renders brokered notifications for crew and guests with
  read-only Signal K access.

## Consequences

Historical small-screen clients may remain available for reference, but they
should not receive new feature work except urgent fixes. New small-screen alert
behaviour belongs in AJRM Marine Alerts.

Read-only Signal K access becomes the intended permission level for crew alert
viewers. Future guest setup can issue read-only tokens without granting profile,
silence or notification-management writes.

## Follow-up

- Keep AJRM Marine Alerts read-only from the vessel-control point of view.
- Move any remaining operational settings out of viewer apps and into their
  owning plugins.
