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
| [server](https://github.com/OpenHDO/server) | C++ control plane, API, persistence, orchestration, CLI, and panel host |
| [server-dashboard](https://github.com/OpenHDO/server-dashboard) | Configurable dashboard module |
| [server-logic](https://github.com/OpenHDO/server-logic) | Flows, nodes, conditions, and actions |
| [server-linker](https://github.com/OpenHDO/server-linker) | Server-side Linker sessions, inventory, health, and routing |
| [linker](https://github.com/OpenHDO/linker) | Standalone hardware-access process near radios and devices |
| [agent](https://github.com/OpenHDO/agent) | Desktop and edge operating-system capabilities |
| [sdk](https://github.com/OpenHDO/sdk) | Generated bindings, SDK tooling, and compatibility helpers |
| [app](https://github.com/OpenHDO/app) | Native and specialized client applications |
| [get-started](https://github.com/OpenHDO/get-started) | Installation, first run, and contributor onboarding |

## Ownership and authority

The repository named below is the decision-maker for the responsibility in its
row. A consumer may propose a change, but it must not silently fork the owning
repository's contract or vocabulary.

| Repository | Owns | Does not own |
| --- | --- | --- |
| `about` | Product overview, system boundaries, shared vocabulary, ADR process, and cross-repo governance | Runtime code, database schema, or wire-format implementation |
| `server` | Canonical server API and Linker protocol schemas under `contracts/<major>/`; server state, policy, persistence, orchestration, and module hosting | Device-driver code, client-specific UI, or generated SDK bindings |
| `server-dashboard` | Dashboard module behavior and panel-facing presentation | Canonical server state or protocol schemas |
| `server-logic` | Flow, node, condition, and action behavior | Transport, device access, or direct database access |
| `server-linker` | Server-side Linker sessions, inventory, health, and routing implementation | The standalone Linker process or device-driver implementation |
| `linker` | Standalone hardware access, transports, discovery, pairing, and device drivers | Canonical server state or policy |
| `agent` | Desktop and edge operating-system capabilities and their permissions | Hardware ownership or server persistence |
| `sdk` | Generated language bindings, SDK tooling, and compatibility helpers derived from published contracts | The normative wire schemas |
| `app` | Native and specialized client applications | Server orchestration or canonical data |
| `get-started` | Installation, first-run, and contributor onboarding | Product or protocol authority |

When ownership is unclear, [ARCHITECTURE.md](ARCHITECTURE.md) is the starting
point; the runtime owner is then recorded in an ADR before a public contract
changes.

## System shape

```text
clients / panel / CLI
          │ public server API
          ▼
   openhdo-server ─── SQLite state and orchestration
          │
          │ versioned Linker protocol
          ▼
   openhdo-linker ─── Wi-Fi / Bluetooth / Zigbee / USB / serial
```

The server owns shared state and policy. A Linker owns local hardware access.
Multiple Linkers can connect to one server, locally or across an authenticated
network. Agents and plugins are separate extension points with explicit
permissions.

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

- C++20/23 for the server runtime, Linker runtime, agents, and native SDK;
- CMake for builds and dependency boundaries;
- SQLite for the first single-node persistence backend;
- HTTP and WebSocket for client and live-update APIs;
- JSON Schema and versioned language-neutral messages at process boundaries;
- React + TypeScript + Tailwind + shadcn-style primitives for web panels;
- Python 3.11+ as an optional SDK and Linker implementation language.

The initial deployment is a single server with zero or more Linkers. Clusters,
cloud dependency, generic microservices, and direct plugin database access are
out of scope until real operational requirements justify them.

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

The [server](https://github.com/OpenHDO/server) repository contains the first
buildable foundation: C++ runtime and CLI, protocol v1 schemas, a panel shell,
Python reference SDK, and CI. The next end-to-end milestone is a real server
API plus one authenticated Linker session.
