# OpenHDO architecture

Current decision (2026-10-09): built-in Devices and server-owned integrations.
This supersedes Linker responsibilities in old ADRs; decision history is retained.

## Ownership

The Python server is a modular monolith. It owns inventory/state, auth, credentials,
persistence, orchestration and its React panel. Devices is core navigation, not a
connector module. Smart-home adapters live inside the server.

Tuya Cloud connects a linked Tuya Smart / Smart Life account and imports actual
devices. Wi-Fi onboarding stays in the vendor app. Home Assistant is outside this
release. `server-dashboard` owns reusable UI, not credentials/state. `server-logic`
executes flows through server capabilities. `agent` owns desktop/OS capabilities.
`sdk` derives transport-free types from contracts. `app` packages a separately built
dashboard.

## Device lifecycle

1. Admin connects the smart-home account in Devices.
2. Server validates credentials, reads inventory and actual function specifications.
3. Stable IDs are integration-scoped; local metadata survives sync/restart.
4. Users send actions constrained by advertised types, ranges and modes.
5. Server sends the vendor command and reads actual cloud state again.
6. Periodic sync reconciles removals and marks failures unavailable.

Cloud acceptance is not hardware confirmation. Empty inventory stays empty.
Unknown formats receive no guessed semantics. Secrets stay in private server data.

## Contracts

APIs use `/api/v1`; schemas/examples belong to `server/contracts/v1`. Clients never
access SQLite/Python internals. Cookie mutations require CSRF. Integration
management is admin-only; ordinary users read/control devices.

Light v1 remains for supported lamps. Legacy `linker_id` holds the integration ID;
old bindings need new IDs. Historical schemas stay for SDK DTO compatibility,
while Linker/discovery/pairing runtime routes and processes are retired.

`linker` and `server-linker` leave the active workspace, with Git history preserved
in a sibling archive. Exact onboarding/checks live in the server repository.
