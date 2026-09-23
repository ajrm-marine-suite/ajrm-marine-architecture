# ADR-020: Add static contract checking without changing the JavaScript runtime

Status: accepted  
Date: 2026-09-18

## Context

AJRM Marine plugins are established Signal K JavaScript packages. A wholesale
TypeScript conversion would create broad build, packaging and deployment change
without itself validating marine-domain assumptions. The suite nevertheless
benefits from earlier detection of misspelled fields, inconsistent return
shapes, nullable values and drift between shared contracts.

The Weather Database pilot showed that TypeScript's `checkJs` mode and JSDoc
types can provide that checking while retaining the existing JavaScript runtime
and Signal K package layout.

## Decision

AJRM Marine will adopt incremental development-time static checking:

- shipped runtime code remains JavaScript unless a later, explicit decision
  authorises a TypeScript build for a particular package;
- a plugin may add a `jsconfig.json` using `allowJs`, `checkJs` and `noEmit`,
  with TypeScript and any supporting type declarations as development-only
  dependencies;
- adoption happens when a plugin is actively changed, beginning with coherent
  modules and shared contract boundaries rather than a suite-wide annotation
  campaign;
- once configured, `npm test` includes `npm run typecheck`, and a release must
  pass both checks;
- production installation must continue to work with development dependencies
  omitted and must not require a compile or post-install build step;
- new or materially changed shared data shapes should receive useful JSDoc
  types, while runtime validation remains mandatory at untrusted API, file,
  provider and cross-plugin boundaries; and
- inferred types are not authority for navigation, tide, weather, collision or
  other marine semantics. New domain assumptions must still be documented,
  tested and reviewed with the skipper when they have not been independently
  verified.

Initial adoption may use a non-strict configuration to expose errors in
manageable groups. Strictness can be increased module by module when the
resulting changes are reviewable and covered by tests.

## Verification

For every package that has adopted this decision:

1. `npm run typecheck` completes without emitting files.
2. `npm test` passes with type checking in its normal path.
3. `npm pack --dry-run` contains the intended runtime and configuration files.
4. The packed package installs with development dependencies omitted.
5. A proportionate Signal K activation and browser/API smoke test passes.

## Consequences

- Contract mistakes can be caught before runtime with no new production
  runtime dependency.
- Work remains incremental and can accompany normal maintenance rather than
  consuming a large one-off conversion budget.
- Some JSDoc and configuration maintenance is added to each adopted package.
- Static checking complements tests and runtime validation; it does not prove
  that a marine calculation or operational assumption is correct.
