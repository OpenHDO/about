# OpenHDO

Open Home Device Orchestration is a self-hosted control plane for devices,
computers, dashboards and automations. HDO contains its own Devices panel.
Users add devices through existing smart-home accounts; Tuya / Smart Life is first.

| Repository | Responsibility |
| --- | --- |
| server | Python runtime/API, auth, devices, integrations and React HDO panel |
| server-dashboard | Reusable dashboard UI, widgets and independent instances |
| server-logic | Flows, nodes, conditions and actions |
| agent | Desktop and OS capabilities |
| sdk | Transport-free types and client contracts |
| app | Packaging a separately built static dashboard |
| get-started | Installation and Tuya onboarding |
| about | Architecture and decision history |

`linker` and `server-linker` are retired. Adapters live inside `server`; no connector
repository or hardware process is needed for Tuya. Historical ADRs record previous
decisions. See [architecture](ARCHITECTURE.md).
