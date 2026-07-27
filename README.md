# AJRM Marine Architecture

Suite-level architecture, contracts, diagrams, and design decisions for the
AJRM Marine Signal K applications.

AJRM Marine is a set of cooperating Signal K plugins and webapps for vessel
traffic awareness, audible alerts, voyage capture, replay, diagnostics,
instrument displays, GPS integrity, dead reckoning, simulation, and built-in
testing.

The architecture rule is simple:

> Backend plugins calculate and publish authoritative state. Frontends render
> that state and send explicit user actions. Signal K remains the public
> integration layer.

## Current Documents

- [AJRM Marine Suite Architecture 2026](docs/ajrm-marine-suite-architecture-2026.md)
- [Compatibility contracts](docs/compatibility-contracts.md)
- [Implementation and verification plan](docs/implementation-plan.md)
- [Pre-public hardening checklist](docs/pre-public-hardening.md)
- [July 2026 voyage software baseline](docs/voyage-software-baseline-2026-07.md)
- [Architecture decisions](docs/decisions/README.md)
- [Target architecture diagram](docs/diagrams/target-architecture.mmd)

## Code Licensing

The AJRM Marine software repositories are licensed under the GNU Affero General
Public License v3.0 or later (`AGPL-3.0-or-later`). Commercial licensing is
available by arrangement for organisations that want different terms.

## Documentation Licensing

The documents and diagrams in this repository are not licensed under AGPL.
They are:

**All rights reserved except for personal/non-commercial Signal K use.**

See [LICENSE](LICENSE) for the permitted use and commercial licensing terms.

## Scope

This repository covers the suite-level behaviour that crosses individual plugin
boundaries:

- Traffic, collision-risk wording, profiles, stationary automute, and voyage
  state.
- Notifications, active/recent alert projections, priority, lifecycle, and
  supersession.
- Audio, Piper rendering, speaker/stream/desktop-player delivery, and audible
  timeline rules.
- Display, Console, Alerts, OpenCPN message display, and other visual clients.
- Capture, Logger, Snapshot, Voyage Viewer, and replay/debug workflows.
- GPS Integrity and DR Plotter, including lost-GPS fallback, independent DR,
  plotted fixes, and diagnostics.
- Own-vessel navigation reference selection, magnetic variation, compass/COG
  fallback, current/leeway provenance, and DR independence.
- Simulator and BITE as repeatable test infrastructure.
- Publication, AppStore readiness, and public repository expectations.

## Authorship

AJRM Marine is authored and maintained by Anthony McDonald, with assistance from
William McAusland. OpenAI Codex has provided substantial assistance with code
generation, refactoring, documentation, and testing.

Signal K is a separate open marine data project. AJRM Marine builds on Signal K
but is not an official Signal K project unless and until accepted by that
community.

## Safety

AJRM Marine is beta/test software. It is intended to assist situational
awareness and software experimentation. It must not be relied upon as the sole
means of navigation, collision avoidance, anchoring safety, GPS integrity
monitoring, or vessel control.
