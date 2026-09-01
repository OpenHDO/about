# ADR-0002: Make Python the primary server runtime and React the web-panel standard

- Status: Accepted
- Date: 2026-09-01
- Owner repository: `about` (cross-repository architecture)
- Affected repositories: `server`, `server-linker`, `linker`, `server-dashboard`, `sdk`, `app`
- Related contract/schema: `server/contracts/<major>/` (published by `server`)
- Surface classification: `Other` (runtime and cross-repository UI architecture)

## Context

The product architecture needs one unambiguous owner for the backend/runtime
and one web-panel technology direction. Earlier wording could be read as
making a native runtime the primary control plane and as allowing vendor details
to leak into server-facing surfaces. The device path must keep the server model
abstract while giving vendor-specific connections and device logic a single
owner.

## Decision

- `server` is the primary Python backend/runtime/API. It owns persistence,
  orchestration, policy, canonical state, versioned contracts, and the built-in
  server admin surface.
- The server exposes an abstract, vendor-neutral `Light` model. Its normalized
  state includes power, brightness, RGB color, and declared capabilities where
  supported. Public API contracts, persistence, SDKs, and web panels must not
  expose vendor names, credentials, discovery details, raw payloads, or vendor
  protocol shapes.
- The Python `linker` owns vendor-specific connections and device logic,
  including discovery, pairing, credentials, protocol drivers, and translation
  to and from the abstract Light model.
- `server-dashboard` is the sole reusable React/TypeScript dashboard module. It
  owns dashboard definitions, visual components, widgets, composition, and
  independently scoped dashboard instances, and consumes the server API.
- The built-in server admin surface remains a separate React panel built into
  and owned by `server`.
- `app` accepts a pre-built static dashboard artifact from an external host/build
  pipeline that uses `server-dashboard`; the library distribution (`dist`) of
  `server-dashboard` itself is not an `app` input. It packages that artifact
  for target platforms and writes metadata consumed by an external platform
  launcher; the `app` repository currently contains no launcher runtime. It has
  no UI, screens, dashboard definitions, server API, or device/light transport
  responsibility.
- C++ is optional and limited to SDK/native integration. The server runtime,
  control-plane implementation, and web panels have no C++ dependency.
- Simulator runtimes, Home Assistant runtimes or APIs, and gateway deployments
  are prohibited as OpenHDO runtime components. Real device access uses the
  Python Linker boundary.

## Boundaries and compatibility

The server remains the authority for normalized Light state and policy. The
Python Linker is the authority for vendor-specific device access. The existing
versioned server and Linker contracts remain the compatibility boundary; any
wire change is additive or breaking under the governance in
[`ARCHITECTURE.md`](../../ARCHITECTURE.md). Vendor-specific fields are not a
shortcut for extending the public model.

## Alternatives considered

- **Native runtime as primary:** rejected because it conflicts with the Python
  server runtime decision and makes optional native integration a required
  foundation.
- **Vendor integrations inside `server`:** rejected because it leaks protocol,
  credential, and failure details into canonical state and public clients.
- **Home Assistant runtime or gateway:** rejected because OpenHDO owns its own
  Linker boundary and must not acquire an external runtime or gateway dependency.
- **Simulator as the runtime path:** rejected because the reference path must
  use a real device through the Python Linker.

## Consequences and evidence

The server and its web panels share a clear Python/React direction; vendor
details stay isolated in one process; C++ integrations can be added without
making the server depend on native toolchains. The app static-artifact packaging
path remains outside the UI, API, and device boundaries. The cost is
maintaining Python Linker drivers and normalized mappings for each supported
vendor.

The decision is evidenced by the synchronized repository map, architecture
contract, and RGB-light vertical slice. Implementation repositories must cover
the abstract Light contract, Linker translation, `server-dashboard` API usage,
malformed input, device unavailability, and reconnect behavior with provider
and consumer contract tests.

## Review and lifecycle

- Reviewers/consumer acknowledgements: Pending in affected repositories
- Decision date: 2026-09-01
- Superseded by: N/A
