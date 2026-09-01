# ADR-0001: Deliver the RGB light vertical slice simulator-first

- Status: Accepted
- Date: 2026-09-01
- Owner repository: `about` (cross-repository architecture)
- Affected repositories: `server`, `server-linker`, `linker`, `server-dashboard`, `app`
- Related contract/schema: `server/contracts/<major>/` (published by `server`)
- Surface classification: `Other` (cross-repository vertical slice)

## Context

OpenHDO needs one complete device path that proves ownership and contracts
before hardware availability becomes a dependency. RGB light is the smallest
useful slice: it has canonical state, commands, replies, events, a transport
boundary, and more than one client surface.

## Decision

- `server` owns canonical RGB light/device state, policy, the versioned public
  contract, and the built-in configuration/admin panel.
- `server-linker` owns the server-side Linker boundary: session negotiation,
  registration, health, inventory, routing, and command/result mapping.
- `linker` owns physical transport and driver behavior. A deterministic RGB
  simulator is the first driver and uses the same boundary as hardware.
- `server-dashboard` is a reusable client-dashboard module; many independently
  scoped dashboard instances may consume the same state. `app` is another
  client surface. Neither is an authority for device state or server admin.

The semantic minimum is a light state containing power, brightness, and RGB
color, plus commands to change it, replies to conclude commands, and events to
report state that changed. Exact wire field names belong in the server-owned
contract, not in this ADR.

## Simulator-first path

1. Define the versioned contract and canonical in-memory state in `server`.
2. Control and inspect the simulated light through the built-in server admin
   panel.
3. Connect the simulator through `server-linker` using the normal Linker
   negotiation, registration, heartbeat, routing, and result path.
4. Verify one `server-dashboard` instance and one `app` surface through the
   public API, then replace only the simulator driver with a physical RGB
   driver.

No client or server change is allowed when the simulator is replaced by
hardware. The simulator must not bypass authentication, validation, routing,
observability, or the server's canonical state.

## Patterns and contract rules

Use Envelope Wrapper, Command Message, Event Message, Request-Reply, and
Correlation Identifier at the public/process boundary. Use a Message Router in
`server-linker`, a Channel Adapter at the Linker driver edge, and Strategy only
where simulator and physical-driver behavior genuinely varies. Receivers must
be idempotent for retryable commands; message IDs and correlation IDs must be
observable. Do not add a broker, Guaranteed Delivery, or a Message Store until
the delivery requirement is demonstrated.

## Home Assistant reuse boundary

Home Assistant is a reference for semantics and protocol ideas, not an
OpenHDO runtime dependency or authority. The following boundary is mandatory:

- Allowed: study or reuse compatible entity/capability semantics, protocol
  ideas, and code only when its license is compatible with the target OpenHDO
  repository and distribution; record provenance and attribution.
- Not allowed: blind copying, copying code without license verification,
  importing Home Assistant as a hidden runtime dependency, coupling the
  server/Linker contract to Home Assistant APIs, or treating Home Assistant's
  state model as canonical without an OpenHDO contract decision.

Any reused code is isolated behind an owned OpenHDO boundary, reviewed for
license compatibility, and covered by the same validation and contract tests as
new code.

## Consequences and evidence

The slice can run without hardware or Home Assistant and proves the swap from
simulator to driver. It also fixes a single authority for state and keeps the
admin panel separate from reusable client dashboards. Evidence is the server,
server-linker, and Linker simulator contract tests, plus client tests for
initial state, command reply, state event, retry/deduplication, malformed input,
and simulator replacement.
