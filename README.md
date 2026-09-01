# OpenHDO

**Open Home Device Orchestration** is a self-hosted, local-first control plane
for devices, computers, operating systems, services, dashboards, and
automations.

This repository is the product-level source of truth. It explains how the
repositories fit together; implementation details belong in the repository
that owns them.

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
| [sdk](https://github.com/OpenHDO/sdk) | Language-neutral contracts and SDK tooling |
| [app](https://github.com/OpenHDO/app) | Native and specialized client applications |
| [get-started](https://github.com/OpenHDO/get-started) | Installation, first run, and contributor onboarding |

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

## Status

The [server](https://github.com/OpenHDO/server) repository contains the first
buildable foundation: C++ runtime and CLI, protocol v1 schemas, a panel shell,
Python reference SDK, and CI. The next end-to-end milestone is a real server
API plus one authenticated Linker session.
