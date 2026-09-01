# OpenHDO architecture

## Product boundaries

OpenHDO is a local-first control plane with explicit process boundaries:

- **Server** owns canonical state, orchestration, policy, persistence, audit,
  the public API, and server modules.
- **Linker** runs close to hardware and owns transports, device discovery,
  pairing, local credentials, protocol drivers, and device events.
- **Agent** exposes desktop and operating-system capabilities under a separate
  identity; it is not a replacement for a hardware Linker.
- **Plugin** is an isolated extension that communicates through public,
  versioned contracts and never accesses the server database directly.
- **Client** is a view or control surface. It does not duplicate orchestration
  or become a second source of truth.

The first server release is a modular monolith. `server-dashboard`,
`server-logic`, and `server-linker` are clear internal module boundaries and
separate repositories, but they are not forced into networked microservices.

## Domain model

The stable domain vocabulary is:

`Device` → a physical or virtual controllable thing; `Entity` → a concrete
value or endpoint; `Capability` → a typed feature; `Event` → a fact that
happened; `Command` → a validated request; `Flow` → triggers, conditions, and
actions; `Location` → a physical or logical place; `Linker` → a hardware-access
process; `Session` → its authenticated server connection.

These concepts are serialized independently of any dashboard layout or SQLite
schema.

## Communication

The public server API is planned as HTTP for queries, configuration, and
commands, plus WebSocket for live state, events, logs, and execution updates.
Process boundaries use small versioned JSON messages. The logical v1 envelope
is defined in the [server contracts](https://github.com/OpenHDO/server/tree/master/contracts/v1);
the concrete transport remains a deployment decision.

Every remote boundary must provide authentication, encryption, bounded message
sizes, timeouts, structured errors, and reconnect behavior. Protocol major
versions are negotiated before payload processing. Unknown optional fields may
be ignored; breaking required changes require a new major contract.

## Linker lifecycle

```text
discover → authenticate → negotiate → register → heartbeat
                                      │
                         inventory / state / events / commands
                                      │
                         reconnect or explicit decommission
```

A Linker registration identifies its version, location, transports, health,
and capabilities. The server turns the reported device inventory into its
canonical registry and routes commands back through the owning session.

## Security model

Security is a platform boundary: least privilege for users, flows, agents,
plugins, and Linkers; authenticated remote access; encrypted transport; input
validation; execution timeouts; crash containment; and audit records for
security-sensitive commands and configuration changes.

The first identity model should stay small and explicit. More advanced
multi-user identity, federation, and policy systems should be added only after
the basic command and capability model is operational.

## Deployment

- **Same host:** server and one or more Linkers on a Raspberry Pi, PC, or
  always-on local machine.
- **Separate hardware host:** server on a PC/VPS and a Linker on a Raspberry Pi
  or gateway with the required radio or USB device.
- **Multiple locations:** several authenticated Linkers connect to one server;
  each retains its own location, transports, inventory, credentials, and health.

There is no mandatory cloud account. Clustering and distributed storage are
deliberately outside the first deployment target.

## Delivery sequence

1. Server foundation, contracts, logging, registry, and in-memory execution.
2. HTTP/WebSocket API, SQLite migrations, authentication baseline, and history.
3. Linker process, one reference transport, identity, heartbeat, and reconnect.
4. Dashboard, Logic, and server-Linker modules in the web panel.
5. Python/C++ SDKs, agents, additional clients, and more integrations.

Each stage should end with one complete vertical path before expanding the
number of modules or supported transports.
