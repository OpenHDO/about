# OpenHDO

**Open Home Device Orchestration** is a self-hosted, local-first control plane
for devices, computers, operating systems, services, dashboards, and
automations.

This repository is the product-level source of truth. It explains how the
repositories fit together, owns the shared product vocabulary, and records
cross-repository architectural decisions. Implementation details and runtime
contracts belong in the repository that owns them.

## Start here

- New users: [get-started](https://github.com/OpenHDO/get-started)
- Control plane: [server](https://github.com/OpenHDO/server)
- Architecture: [ARCHITECTURE.md](ARCHITECTURE.md)
- Hardware access: [linker](https://github.com/OpenHDO/linker)

## Repository map

| Repository | Responsibility |
| --- | --- |
| [server](https://github.com/OpenHDO/server) | Python control plane backend/runtime/API, persistence, orchestration, CLI, and built-in React web admin surface |
| [server-dashboard](https://github.com/OpenHDO/server-dashboard) | Reusable React web-panel module for many dashboard instances |
| [server-logic](https://github.com/OpenHDO/server-logic) | Flows, nodes, conditions, and actions |
| [server-linker](https://github.com/OpenHDO/server-linker) | Python server-side Linker sessions, inventory, health, and routing |
| [linker](https://github.com/OpenHDO/linker) | Standalone Python Linker with vendor-specific connections and device logic |
| [agent](https://github.com/OpenHDO/agent) | Desktop and edge operating-system capabilities |
| [sdk](https://github.com/OpenHDO/sdk) | Generated bindings, Python SDK tooling, and optional C++/native integration SDKs |
| [app](https://github.com/OpenHDO/app) | Native and specialized client applications |
| [get-started](https://github.com/OpenHDO/get-started) | Installation, first run, and contributor onboarding |

## Ownership and authority

The repository named below is the decision-maker for the responsibility in its
row. A consumer may propose a change, but it must not silently fork the owning
repository's contract or vocabulary.

| Repository | Owns | Does not own |
| --- | --- | --- |
| `about` | Product overview, system boundaries, shared vocabulary, ADR process, and cross-repo governance | Runtime code, database schema, or wire-format implementation |
| `server` | Python backend/runtime/API; canonical server API and Linker protocol schemas under `contracts/<major>/`; server state, policy, persistence, orchestration, module hosting, and the built-in React web admin surface | Device-driver code, reusable client-dashboard behavior, or generated SDK bindings |
| `server-dashboard` | Reusable React web-panel module behavior, instance-safe presentation, and its client-facing interaction contract | The built-in server admin surface, canonical server state, or a particular dashboard instance |
| `server-logic` | Flow, node, condition, and action behavior | Transport, device access, or direct database access |
| `server-linker` | Python server-side Linker sessions, inventory, health, and routing implementation | The standalone Linker process or vendor-specific device logic |
| `linker` | Standalone Python hardware access, vendor-specific connections, discovery, pairing, credentials, and device logic | Canonical server state or policy |
| `agent` | Desktop and edge operating-system capabilities and their permissions | Hardware ownership or server persistence |
| `sdk` | Generated language bindings, Python SDK tooling, optional C++/native integration SDKs, and compatibility helpers derived from published contracts | The normative wire schemas or the server runtime |
| `app` | Native and specialized client applications | Server orchestration or canonical data |
| `get-started` | Installation, first-run, and contributor onboarding | Product or protocol authority |

When ownership is unclear, [ARCHITECTURE.md](ARCHITECTURE.md) is the starting
point; the runtime owner is then recorded in an ADR before a public contract
changes.

## System shape

```text
server-dashboard / app / panel / CLI
                 │ public server API
                 ▼
           openhdo-server ─── Python backend/runtime/API, abstract Light model
                 │
                 │ versioned Linker protocol
                 ▼
       server-linker boundary
                 │
                 ▼
       Python openhdo-linker ─── vendor-specific Wi-Fi driver ─── RGB light
```

The server owns shared state and policy. Its public contracts expose an abstract,
vendor-neutral Light model (including normalized power, brightness, color, and
capabilities where supported). Vendor names, protocols, credentials, discovery
details, and raw payloads stay hidden inside the Python Linker; they must not
leak into server state, the public API, SDKs, or React web panels. A Linker owns
local hardware access and vendor-specific device logic. Multiple Linkers can
connect to one server, locally or across an authenticated network. Agents and
plugins are separate extension points with explicit permissions.

## RGB light vertical slice

The first end-to-end path uses one real vendor-specific RGB light over Wi-Fi:

- `server` is the Python backend/runtime/API. It owns the abstract canonical
  Light model, server admin surface, public API, commands, replies, and state
  events.
- `server-linker` is the Python server-side Linker session, registration,
  health, inventory, routing, and result-mapping boundary.
- `linker` is a Python process that owns the vendor-specific connection and
  device logic for the real light.
- `server-dashboard`, the built-in admin surface, and any other web panels are
  React clients of the server API; neither web panels nor `app` owns canonical
  state or vendor details.

The complete path is recorded in [ADR-0001](docs/ADR/0001-rgb-light-vertical-slice.md).

## Dashboard instances and server admin

A **Dashboard instance** is one configured use of the reusable
`server-dashboard` client module. Many instances may coexist: for example, a
main dashboard, an embedded wall panel, a room-specific dashboard, or a
setup-specific dashboard. The module is reusable; the instance supplies its
own identity, placement, context, layout, and permissions through public
contracts.

The **server admin surface** is different. It is a privileged, built-in React
web panel shipped and owned by `server` for server configuration, policy,
identities, Linker administration, and operational controls. It is not a
`server-dashboard` instance and must not be moved into the reusable client
module. See [ARCHITECTURE.md](ARCHITECTURE.md) for the ownership and pattern
rules.

## Protocol vocabulary

Use these terms consistently in issues, schemas, logs, and tests. The detailed
boundary rules live in [ARCHITECTURE.md](ARCHITECTURE.md).

| Term | Meaning |
| --- | --- |
| Envelope | Versioned metadata around a payload: contract, message kind, message ID, timestamp, sender, and optional correlation ID. |
| Command | An addressed, validated request to change state or perform work; it may be retried and must have an explicit result. |
| Event | An immutable fact that already happened; consumers may observe it but cannot alter its meaning. |
| Query | A read-only request for current or historical data. |
| Reply | The response to a command or query; it carries success or a structured error and echoes correlation. |
| Correlation ID | The identifier joining a request, reply, retries, logs, and resulting events. |
| Message ID | The unique identifier used for deduplication and tracing of one message delivery. |
| Capability | A typed operation or value a device/entity exposes; it is not a transport or UI control. |
| Contract version | The negotiated major version plus schema revision governing a boundary. |

Commands express intent; events express facts; queries read; replies conclude a
request. Do not use “event” for a command or “message” as an unqualified
substitute when the distinction affects delivery, authorization, or retries.

## Naming

The standalone hardware-access process is **OpenHDO Linker**:

- repository/process role: `linker`;
- executable: `openhdo-linker`;
- server-side module: `server-linker`;
- administration CLI: `linkercli`.

`Linker` is preferred over `connector`: it describes the component's job while
leaving “adapter” available for individual device drivers. User-facing names
include `OpenHDO` where the word could be confused with a compiler linker.

## Technology direction

- Python 3.11+ for the server backend/runtime/API and Linker;
- SQLite for the first single-node persistence backend;
- HTTP and WebSocket for client and live-update APIs;
- JSON Schema and versioned language-neutral messages at process boundaries;
- React + TypeScript + Tailwind + shadcn-style primitives for every web panel;
- C++ is limited to optional SDK/native integration; the server runtime and web
  panels have no native dependency.

The initial deployment is a single server with zero or more Linkers. Clusters,
cloud dependency, generic microservices, and direct plugin database access are
out of scope until real operational requirements justify them.

Simulator runtimes, Home Assistant runtimes/APIs, and gateway deployments are
not OpenHDO runtime components. Real device access goes through the Python
Linker, with vendor-specific details isolated at its driver edge.

## Engineering guardrails

- Every public boundary is versioned, schema-validated, observable, and tested
  by both its owner and its consumers.
- Prefer an in-process module boundary in the modular monolith. Introduce a
  process, queue, or broker only when isolation, failure, deployment, or
  ownership requirements justify its operational cost.
- Keep authority singular: the server owns canonical state and policy; clients,
  Linkers, agents, and plugins report or request through contracts.
- Record a cross-repository contract change and any major pattern choice in an
  ADR before implementation. See [ARCHITECTURE.md](ARCHITECTURE.md) and the
  [ADR template](docs/ADR/0000-template.md).

## Status

The [server](https://github.com/OpenHDO/server) repository contains the current
Python backend/runtime/API foundation, protocol v1 schemas, a React panel shell,
Python SDK tooling, and CI. The runtime decision is recorded in
[ADR-0002](docs/ADR/0002-python-server-react-panels.md). The next end-to-end
milestone is the real RGB Wi-Fi path with one authenticated Python Linker
session.
