# OpenHDO architecture

This document is the cross-repository architecture contract. It defines
ownership, vocabulary, boundary rules, and the minimum process for decisions.
It does not replace implementation documentation in the owning repository.

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

## Repository ownership

Ownership means the repository is accountable for the behavior, compatibility,
tests, and release notes of that responsibility.

| Repository | Canonical responsibility | Boundary rule |
| --- | --- | --- |
| `about` | Product vocabulary, repository map, architecture, ADR process, and cross-repo governance | Documents decisions; owns no runtime state or wire schema. |
| `server` | Server API, Linker protocol schemas in `contracts/<major>/`, canonical state, policy, persistence, and orchestration | Publishes normative schemas and compatibility decisions. |
| `server-dashboard` | Dashboard module and its presentation behavior | Uses server APIs; never becomes a source of truth. |
| `server-logic` | Flows, nodes, conditions, and actions | Requests server capabilities; never writes the database directly. |
| `server-linker` | Server-side Linker sessions, inventory, health, and routing | Implements the server side of the Linker contract. |
| `linker` | Hardware access process, transports, discovery, pairing, and drivers | Implements device access behind the Linker contract. |
| `agent` | Desktop and edge OS capabilities | Uses a separate identity and explicit permissions. |
| `sdk` | Generated bindings and SDK tooling | Derives from published schemas; cannot redefine them. |
| `app` | Native and specialized clients | Consumes public contracts; does not duplicate orchestration. |
| `get-started` | Installation and onboarding | Documents released behavior; does not define it. |

The owner reviews changes to its responsibility and supplies compatibility
evidence. Consumers own their migrations and must report incompatibilities;
they do not patch around a contract by inventing a private variant.

## Domain model

The stable domain vocabulary is:

`Device` → a physical or virtual controllable thing; `Entity` → a concrete
value or endpoint; `Capability` → a typed feature; `Event` → a fact that
happened; `Command` → a validated request; `Flow` → triggers, conditions, and
actions; `Location` → a physical or logical place; `Linker` → a hardware-access
process; `Session` → its authenticated server connection.

These concepts are serialized independently of any dashboard layout or SQLite
schema.

## Protocol vocabulary

The following terms are normative across repositories:

- **Envelope**: the versioned wrapper around a payload. It contains at least
  `contract`, `kind`, `message_id`, `occurred_at`, `sender`, and, where a
  message participates in a request, `correlation_id`.
- **Command**: a validated, authorized request to perform work or change state.
  A command has one logical intent and an explicit result; retry behavior is
  part of its contract.
- **Event**: an immutable fact that has occurred. Events are notifications, not
  instructions, and consumers must tolerate redelivery.
- **Query**: a read-only request for current or historical data.
- **Reply**: the result of a command or query, with either a typed success
  payload or a structured error, and the request's correlation identifier.
- **Message ID**: the unique identity of one message; receivers use it for
  tracing and deduplication.
- **Correlation ID**: the identity shared by a request, reply, retries, logs,
  and resulting events.
- **Capability**: a typed operation or value exposed by an entity. A transport,
  UI control, and capability are different things.
- **Contract version**: the negotiated major version and schema revision for a
  boundary. A schema revision may add compatible optional data; a breaking
  change requires a new major.

Use **command** for intent, **event** for fact, **query** for read, and
**reply** for completion. “Message” is the generic term only when the kind is
irrelevant.

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

### Boundary minimum

Every public HTTP, WebSocket, process, plugin, or SDK boundary must have:

1. a named owner and a negotiated major version;
2. a machine-readable schema and validation on ingress and egress;
3. stable message IDs, correlation IDs where applicable, structured errors,
   size/time limits, and explicit timeout/reconnect semantics;
4. authorization at the operation/capability level;
5. logs, metrics, and traces that do not disclose secrets; and
6. provider tests plus consumer contract tests, including malformed input,
   unknown optional fields, retries, and version mismatch behavior.

The concrete transport may change without changing the logical contract. Keep
transport-specific details in the owning repository's adapter or gateway.

## Pattern selection rules

Patterns are names for a demonstrated responsibility or failure mode, not a
required vocabulary for every component. Choose the smallest fitting pattern,
state the problem it solves, and test its failure semantics. Do not add a
broker, abstraction layer, or asynchronous flow solely to use a pattern name.

| Problem | Default pattern | Use it when | Required evidence / boundary |
| --- | --- | --- | --- |
| Stable metadata around different payloads | EIP Message / Envelope Wrapper; Fowler DTO | A payload crosses a public or process boundary | Validate the envelope and payload independently; preserve version and IDs. |
| Intent, fact, read, and completion | EIP Command Message, Event Message, Request-Reply | The distinction changes authorization, retry, or ownership | Commands are idempotency-aware; events are immutable; replies carry correlation and structured errors. |
| Matching a response to work | EIP Correlation Identifier | Work can be asynchronous, retried, or observed | Propagate it through logs, replies, and resulting events. |
| Choosing a destination | Message Router | More than one receiver or Linker is possible | Centralize routing policy and make the selected destination observable. |
| Routing by validated data | Content-based Router | Destination depends on capability, location, or message content | Validate before routing; keep policy in the server. |
| Dropping irrelevant or unauthorized work | Message Filter | A message is valid but not applicable or permitted | Record the reason; do not silently discard malformed messages. |
| Retry and failure isolation | Idempotent Receiver; Dead Letter Channel | Delivery can repeat or a message can fail permanently | Define deduplication key, retry budget, operator visibility, and replay policy. |
| Durable delivery | Guaranteed Delivery with a Message Store | Losing a command/event is unacceptable and durable storage is available | Document persistence, ordering, retention, backpressure, and recovery; do not promise delivery from memory. |
| Operational commands and health | Control Bus | Operators need bounded control/telemetry over modules or sessions | Keep it separate from domain commands and permission it explicitly. |
| External protocol/device integration | Channel Adapter; Messaging Bridge | A transport or protocol must connect without leaking its shape inward | Translate at the edge; use a bridge only when both sides must vary independently. |
| Server application boundary | Fowler Service Layer + Domain Model | A request coordinates policy and domain behavior | Keep domain rules out of controllers and transports. |
| SQLite persistence | Repository + Data Mapper + Unit of Work | Domain objects must stay independent of storage and writes need a transaction | Modules use interfaces; only the server persistence owner touches the database. |
| External system access | Fowler Gateway; Value Object; Separated Interface | An external resource, identifier, or module boundary needs a stable seam | Keep credentials and wire types at the edge; make invalid values unrepresentable where practical. |
| Real runtime variability | Strategy, Adapter, Bridge, or Plugin | Algorithms, drivers, or extensions genuinely vary independently | Define a narrow interface, lifecycle, permissions, and tests for each implementation. |
| Notifications and lifecycle | Observer or State | Multiple observers need updates, or a lifecycle has valid transitions | Bound subscriptions and enumerate legal state transitions. |
| Ordered policy steps | Chain of Responsibility | Independent validators/policies may accept, reject, or pass work in order | Make order explicit and stop on authorization failure. |
| Coarse client surface | Facade | A client needs a stable, coarse-grained operation over several services | Keep it thin; it must not become a second domain model. |

Before selecting a pattern, answer: what responsibility varies, who owns it,
what failure must be contained, and what test proves the choice? If those
answers are not concrete, use a direct function, interface, or in-process call.

## ADR process

Use an ADR for a cross-repository contract, a new process or persistence
boundary, a durable delivery/retry guarantee, a security model change, or a
pattern choice whose consequences will constrain another repository. Small
local refactors do not need one.

1. Copy [`docs/ADR/0000-template.md`](docs/ADR/0000-template.md) to the next
   number and write it before implementation is merged.
2. Name the decision in imperative terms, list the owning repository and
   affected consumers, and capture rejected alternatives and operational cost.
3. Link the relevant contract/schema, compatibility and migration plan,
   observability, security impact, and test evidence.
4. Review in the owner repository and notify every affected consumer. A
   cross-repo change is not accepted until each consumer has an implementation
   or an explicit migration issue.
5. Mark the ADR `Accepted`, `Rejected`, or `Superseded`; do not rewrite an
   accepted decision. Add a new ADR when the decision changes.

ADRs are architectural records, not a substitute for the contract itself. The
normative schema stays with its owner; this repository records why and how
repositories must coordinate.

## Cross-repository contract governance

`server` owns the normative public and Linker schemas under
`contracts/<major>/`. `server-linker` and `linker` implement the two sides;
`sdk` derives bindings and tooling; clients, agents, plugins, and modules are
consumers. `about` owns this governance and the shared vocabulary, not schema
files.

For every contract change:

1. Open a change proposal in the owning repository and link an ADR when the
   change meets the ADR criteria.
2. Classify it as additive, behavioral, deprecated, or breaking. Additive
   changes use optional fields/defaults and preserve existing meanings;
   changing required fields, meanings, authorization, delivery, or error
   semantics is breaking and requires a new major version.
3. Update the owner schema, examples, provider validation, compatibility
   matrix, and contract tests. Regenerate SDK artifacts only from the
   published schema.
4. Have each consumer run its contract tests and record migration/rollout
   order. During a transition, the server may negotiate and serve two majors;
   it must not guess a payload's version.
5. Publish deprecation and removal dates, telemetry for remaining old clients,
   and rollback/replay behavior before removing a major version.

No repository may make a public boundary change by editing a copied schema,
relying on undocumented fields, or treating a database table as an API. A
contract is ready only when it is versioned, validated, observable, tested,
authorized, and owned.

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

## Pattern references

- [Enterprise Integration Patterns](https://www.enterpriseintegrationpatterns.com/patterns/messaging/index.html)
- [Martin Fowler's enterprise application patterns](https://martinfowler.com/eaaCatalog/)
- [Refactoring.Guru design patterns](https://refactoring.guru/design-patterns/catalog)
