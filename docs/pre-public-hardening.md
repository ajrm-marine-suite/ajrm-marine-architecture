# AJRM Marine Pre-Public Hardening

This checklist records the public-release expectations for AJRM Marine Signal K
plugins before broader announcement or npm/AppStore publication.

## Public Repository Rule

Runtime apps live as independent public repositories under the
`ajrm-marine-suite` GitHub organisation. Each package should be installable on
its own, but Console remains the suite entry point and help surface.

The old personal repositories remain historical references. New fixes should be
made in the public AJRM repositories unless explicitly working on a legacy
reference.

## Active Public Apps

| Package | Public app |
| --- | --- |
| `signalk-ajrm-marine-console` | AJRM Marine Console |
| `signalk-ajrm-marine-traffic` | AJRM Marine Traffic |
| `signalk-ajrm-marine-display` | AJRM Marine Display |
| `signalk-ajrm-marine-notifications` | AJRM Marine Notifications |
| `signalk-ajrm-marine-audio` | AJRM Marine Audio |
| `signalk-ajrm-marine-alerts` | AJRM Marine Alerts |
| `signalk-ajrm-marine-capture` | AJRM Marine Capture |
| `signalk-ajrm-marine-logger` | AJRM Marine Logger |
| `signalk-ajrm-marine-voyage-viewer` | AJRM Marine Voyage Viewer |
| `signalk-ajrm-marine-snapshot` | AJRM Marine Snapshot |
| `signalk-ajrm-marine-gps-integrity` | AJRM Marine GPS Integrity |
| `signalk-ajrm-marine-dr-plotter` | AJRM Marine DR Plotter |
| `signalk-ajrm-marine-simulator` | AJRM Marine Simulator |
| `signalk-ajrm-marine-harbour-editor` | AJRM Marine Harbour Editor |
| `signalk-ajrm-marine-vessel-database` | AJRM Marine Vessel Database |
| `signalk-ajrm-marine-instruments` | AJRM Marine Instruments |
| `signalk-ajrm-marine-instrument-alerts` | AJRM Marine Instrument Alerts |
| `signalk-ajrm-marine-pi-controller` | AJRM Marine Pi Controller |
| `ajrm-marine-audio-player` | AJRM Marine Audio Player |

## Signal K AppStore Readiness

Signal K AppStore visibility depends on npm publication with the appropriate
Signal K keywords. Each plugin/webapp package should have:

- `signalk-node-server-plugin` and/or `signalk-webapp` keywords;
- suitable `signalk-category-*` keywords;
- `signalk.displayName`;
- `signalk.appIcon`;
- at least one `signalk.screenshots` entry;
- `description`, `author`, `repository`, `bugs`, `homepage`, `files`, and
  `license`;
- a `CHANGELOG.md` or matching GitHub release notes;
- a test suite that is safe under the Signal K plugin registry harness.

The Signal K plugin registry scores install, load, activation, schema, tests,
audit, changelog, and screenshots. It does not require a specific permissive
licence.

## Licence Expectations

Runtime code repositories use `AGPL-3.0-or-later` with a commercial licensing
statement.

The architecture repository uses a separate documentation licence: all rights
reserved except for personal/non-commercial Signal K use. Do not copy the full
restricted architecture documents into plugin packages unless the separate
documentation licence is clearly preserved.

## Documentation Cleanup

- Public docs should use AJRM Marine names.
- Old AIS Plus and Watchkeeper names should appear only in migration/history
  notes where needed.
- Avoid boat-specific examples such as private hostnames or yacht names unless
  clearly marked as test evidence.
- Include the standard safety warning that AJRM Marine is beta/test software
  and must not be relied upon as the sole safety or navigation system.
- Acknowledge OpenAI Codex assistance separately from authorship.

## Verification

Before broad publication:

1. Run each package's local test suite.
2. Run `npm pack --dry-run` for each package.
3. Confirm the latest GitHub Actions plugin CI is green.
4. Install the suite on a fresh Signal K server.
5. Run Console BITE, including optional-plugin checks where available.
6. Capture a simulated voyage and confirm Voyage Viewer summaries are useful.
7. Confirm screenshots and README examples match the current UI.
