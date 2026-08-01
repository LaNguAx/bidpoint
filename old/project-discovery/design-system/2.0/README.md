# BidPoint design knowledge base 2.0

Status: Canonical
Last validated: 2026-08-01
Supersedes: [design-system 1.0](../1.0/README.md)

2.0 is a **lean revision**, not a rewrite. It changes the technology stack, the pin policy, the delivery sequence, and how many deployables the ownership rules are distributed across.

It does not change the **domain, the ownership rules themselves, or the messaging semantics** — write authority, transactional outboxes, at-least-once delivery, the close-and-order sequence, and the Kafka-versus-RabbitMQ distinction are the strongest part of 1.0 and are inherited unchanged. ADR-038 reduces seven deployables to four by folding `payment` and `realtime` into `core-platform` as modules; every retained boundary keeps its authority, and no owner gained the right to write another's data.

## What 2.0 contains

| Document | Purpose |
| --- | --- |
| [01 Project thesis](01-project-thesis.md) | Revised purpose, dual goal, and the two-phase framing. |
| [02 Technology stack](02-technology-stack.md) | Local and remote inventory, merged and reduced. |
| [03 Delivery roadmap](03-delivery-roadmap.md) | Resequenced so bidding correctness arrives early. |
| [04 Decision delta](04-decision-delta.md) | Every change from 1.0, with rationale and consequence. |
| [05 Exclusions and open questions](05-exclusions-and-open-questions.md) | Revised register; the stack reduction moved a dozen tools to Excluded. |
| [99 Handoff](99-handoff.md) | Standalone entry point for a new developer or AI session. |

## What 2.0 inherits from 1.0 unchanged

These documents remain the canonical contract. 2.0 does not restate them, and nothing in 2.0 overrides them:

| Inherited document | Covers |
| --- | --- |
| [1.0/02 Product domain and capabilities](../1.0/02-product-domain-and-capabilities.md) | Marketplace behavior, lifecycles, invariants. |
| [1.0/03 Architecture overview](../1.0/03-architecture-overview.md) | Deployables, front doors, end-to-end flows. |
| [1.0/04 Core platform modular monolith](../1.0/04-core-platform-modular-monolith.md) | The five Spring Modulith modules. |
| [1.0/05 Microservice boundaries](../1.0/05-microservice-boundaries.md) | Why each service exists and what it must not own. |
| [1.0/06 Data ownership and consistency](../1.0/06-data-ownership-and-consistency.md) | Write authority, transactions, projections, recovery. |
| [1.0/07 REST, SSE, and service communication](../1.0/07-rest-sse-and-service-communication.md) | Synchronous APIs, discovery, live updates. |
| [1.0/08 Kafka events and transactional outbox](../1.0/08-kafka-events-and-transactional-outbox.md) | Durable facts, envelopes, ordering, replay. |
| [1.0/09 RabbitMQ jobs and notifications](../1.0/09-rabbitmq-jobs-and-notifications.md) | Targeted work, retries, DLQ. |
| [1.0/10 Security and identity](../1.0/10-security-and-identity.md) | Keycloak, JWT validation, authorization, secrets. |
| [1.0/15 Testing and quality](../1.0/15-testing-and-quality.md) | Invariant, boundary, container, and load test strategy. |
| [1.0/23 Primary sources](../1.0/23-primary-sources.md) | What counts as acceptable evidence. |

Where 1.0 documents 11–14 and 16–22 conflict with 2.0, **2.0 wins**; consult them for background rationale only.

## The change in one paragraph

1.0 was a correct architecture wrapped in more platform surface than one engineer can carry to completion. 2.0 keeps the architecture and cuts the surface. Two passes: the first removed tooling that taught nothing (Nx, duplicated telemetry) and relaxed bleeding-edge pins; the second cut the stack from roughly forty tools to twenty by asking of each one whether a backend engineer is ever actually asked about it — which removed the mesh, GitOps, operator, and progressive-delivery tier, dropped deployables from seven to four, and made ECS Fargate the AWS target so that 00 in credits is enough. Delivery is resequenced so bidding concurrency arrives at A3 and real AWS experience at A4, rather than at stage eleven.

## Interpretation

The status vocabulary from 1.0 is retained without change: **Canonical**, **Staged**, **Optional comparison**, **Excluded**, **Open**. A status is a design position, never a claim that code, dependencies, manifests, or infrastructure exist.

Repository reality is unchanged from 1.0: this is a documentation-only discovery repository. The adjacent [structure template](../../structure-template/) contains 788 zero-byte placeholder files and is not an initialized workspace.

## Reading paths

- **First contact or new chat:** [99 Handoff](99-handoff.md), then [04 Decision delta](04-decision-delta.md).
- **Coming from 1.0:** [04 Decision delta](04-decision-delta.md) alone tells you what moved.
- **Starting implementation:** [03 Delivery roadmap](03-delivery-roadmap.md), then [02 Technology stack](02-technology-stack.md), then the inherited 1.0 documents for the stage you are building.
