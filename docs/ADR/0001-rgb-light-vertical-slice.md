# ADR-0001: Deliver the RGB light vertical slice over real Wi-Fi

- Status: Accepted
- Date: 2026-09-01
- Owner repository: `about` (cross-repository architecture)
- Affected repositories: `server`, `server-linker`, `linker`, `server-dashboard`, `app`
- Related contract/schema: `server/contracts/<major>/` (published by `server`)
- Surface classification: `Other` (cross-repository vertical slice)

## Context

OpenHDO needs one complete device path that proves ownership, contracts, and
physical device access. An RGB light is the smallest useful slice: it has
canonical state, commands, replies, events, a transport boundary, and more
than one client surface. The path must exercise a real vendor-specific Wi-Fi
driver.

## Decision

- `server` owns canonical RGB light/device state, policy, the versioned public
  contract, the built-in server admin surface, and the public API.
- `server-linker` owns the server-side Linker boundary: session negotiation,
  registration, health, inventory, routing, and command/result mapping.
- `linker` is a Python 3.11+ process. It owns physical Wi-Fi access and the
  vendor-specific RGB driver, including vendor discovery, authentication, and
  protocol translation.
- `server-dashboard` is the sole reusable React/TypeScript dashboard module. It
  owns dashboard definitions, visual components, widgets, composition, and
  independently scoped dashboard instances; it consumes the same server state.
- `app` is only a packager, launcher, and build wrapper for already-built
  `server-dashboard` artifacts. It has no UI, screens, dashboard definitions,
  server API, or device/light transport responsibility.

The semantic minimum is a light state containing power, brightness, and RGB
color, plus commands to change it, replies to conclude commands, and events to
report state that changed. Exact wire field names belong in the server-owned
contract, not in this ADR.

## Real Wi-Fi path

1. `server` defines the versioned contract and canonical state, exposes the
   public API, and provides the built-in server admin surface.
2. `server-linker` authenticates the Python Linker process, negotiates the
   Linker protocol, registers the device, tracks health, and routes commands.
3. The Python `linker` process discovers and authenticates to the real RGB
   light over Wi-Fi, then translates its vendor protocol at the driver edge.
4. `server-dashboard` reads state and issues commands through the server API;
   it does not connect to the light or become a source of truth. The built-in
   admin surface remains in `server`.
5. `app` only packages or launches an already-built `server-dashboard` artifact;
   it is not part of the server API or device/light transport path.

All commands and reported state cross the normal authenticated, validated,
observable `server-linker` boundary. The server remains canonical when the
device is unavailable, reconnects, or reports a failure.

## Patterns and contract rules

Use Envelope Wrapper, Command Message, Event Message, Request-Reply, and
Correlation Identifier at the public/process boundary. Use a Message Router in
`server-linker` and a Channel Adapter at the Linker driver edge. Receivers must
be idempotent for retryable commands; message IDs and correlation IDs must be
observable. Do not add a broker, Guaranteed Delivery, or a Message Store until
the delivery requirement is demonstrated.

## Vendor source reuse

Source from a specific Home Assistant vendor integration may be adapted only
when its provenance and license are verified as Apache-2.0-compatible. Preserve
the required copyright, license, and attribution notices in the owning
repository. Adapted code must be isolated in the Python Linker driver edge;
OpenHDO must not import a Home Assistant runtime or depend on its APIs.

## Consequences and evidence

The slice validates the real Wi-Fi path, keeps canonical state and policy in
the server, and keeps the dashboard and admin surfaces independent of vendor
protocol details. Evidence is the server, server-linker, and Python Linker
contract coverage, plus dashboard/admin tests for initial state, command reply,
state event, retry/deduplication, malformed input, device unavailability, and
reconnect behavior.
