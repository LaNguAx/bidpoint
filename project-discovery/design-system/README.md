# BidPoint design system

Status: Canonical
Last validated: 2026-08-01

## Platform and engineering reading order

11. [Local development platform](11-local-development-platform.md)
12. [AWS production platform](12-aws-production-platform.md)
13. [CI/CD and GitOps](13-ci-cd-and-gitops.md)
14. [Observability](14-observability.md)
15. [Testing and quality](15-testing-and-quality.md)
16. [Monorepo and repository structure](16-monorepo-and-repository-structure.md)
17. [Local technology stack](17-local-technology-stack.md)
18. [Remote technology stack](18-remote-technology-stack.md)
19. [Versions and compatibility](19-versions-and-compatibility.md)
20. [Delivery and learning roadmap](20-delivery-and-learning-roadmap.md)

`design-system` is BidPoint's canonical product and architecture knowledge base. It is not a frontend component library. These documents describe a target design for a documentation-only discovery repository; they do not describe an existing implementation.

## Recommended reading order

1. [Project thesis](01-project-thesis.md) ? purpose, audience, learning outcomes, and scope boundaries.
2. [Product domain and capabilities](02-product-domain-and-capabilities.md) ? marketplace behavior, lifecycles, and invariants.
3. [Architecture overview](03-architecture-overview.md) ? deployables, front doors, platform roles, and end-to-end flows.
4. [Core platform modular monolith](04-core-platform-modular-monolith.md) ? the five Spring Modulith modules and their package contract.
5. [Microservice boundaries](05-microservice-boundaries.md) ? why each surrounding service exists and what it must not own.
6. [Data ownership and consistency](06-data-ownership-and-consistency.md) ? write authority, transaction boundaries, projections, and recovery.
7. [REST, SSE, and service communication](07-rest-sse-and-service-communication.md) ? synchronous APIs, Kubernetes discovery, and live updates.
8. [Kafka events and transactional outbox](08-kafka-events-and-transactional-outbox.md) ? durable facts, event envelopes, ordering, and replay.
9. [RabbitMQ jobs and notifications](09-rabbitmq-jobs-and-notifications.md) ? targeted work, delivery attempts, retries, and DLQ handling.
10. [Security and identity](10-security-and-identity.md) ? Keycloak, JWT validation, authorization, secrets, and threat boundaries.

## Interpretation

Unless a section is explicitly marked **Staged**, **Open**, **Optional**, or **Excluded**, it is part of the canonical target. Staged capabilities are intentional later increments, not implied day-one implementation. Open choices remain undecided until evidence and a separate decision record justify selecting them.
