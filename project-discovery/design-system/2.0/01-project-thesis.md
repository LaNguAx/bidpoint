# BidPoint — A Distributed Systems Reference Platform (2.0)

Status: Canonical
Last validated: 2026-08-01
Supersedes: [1.0/01 Project thesis](../1.0/01-project-thesis.md)

## Thesis

BidPoint is a deliberately bounded auction marketplace used to practise production-grade backend engineering and distributed-systems behavior. The marketplace is credible enough to make correctness matter, yet intentionally avoids domain complexity that would hide the engineering lessons. This repository is documentation-only: the architecture is a target design, not an account of software that already exists.

1.0 stated the thesis correctly and then buried it. The auction concurrency work — the reason the domain was chosen — sat behind a build-tooling stage and a full Kubernetes platform stage. 2.0 states the thesis identically and protects it by sequence.

## Stated goal

This project serves two goals at once, and they are recorded here because they arbitrate every technology decision:

1. **Employability as a backend engineer**, primarily in Spring/Java ecosystems where distributed systems, Kubernetes, and AWS appear in job requirements.
2. **Genuine understanding.** Working as a backend developer without a practical grasp of how distributed systems actually fail is the gap this project closes.

These goals agree far more than they conflict. Both are served by depth on correctness and both are harmed by breadth for its own sake. Where they diverge, the divergence is recorded explicitly in [04 Decision delta](04-decision-delta.md) rather than resolved silently.

## Learning outcomes

Unchanged from 1.0. The target design creates concrete opportunities to learn:

- service and module ownership, with behavior exposed through APIs and events rather than shared tables;
- local ACID transactions combined with transactional outboxes and at-least-once delivery;
- concurrency, ordering, and idempotency where many users bid on the same auction;
- appropriate use of REST, SSE, Kafka, and RabbitMQ according to message purpose;
- derived projections, observable lag, poison-message handling, replay, and partial-failure recovery;
- OAuth2/OIDC identity, independently enforced resource-server security, object-level authorization, and secret isolation;
- production concerns such as hot records, hot Kafka partitions, bounded retries, audit trails, load tests, and failure injection.

Every technology and deployable must teach one of these responsibilities. A split that exists only to make the system look distributed is a design failure.

**2.0 adds one enforcement rule.** The list above is not decorative. A component that cannot be mapped to a specific line is removed, not staged. This rule is what removes Nx and the duplicated remote telemetry stack; both were Canonical in 1.0 and neither maps to any line.

## Two phases

1.0 presented eleven roadmap stages as one continuous march, which is why it reads as unfinishable. 2.0 splits it:

| Phase | Contains | Question it answers | Standalone? |
| --- | --- | --- | --- |
| **A — Correctness** | The marketplace on local k3d, bidding concurrency, an early AWS tracer bullet, outbox, Kafka facts, SSE, the Kafka-to-RabbitMQ fan-out, orders and payment, observability and load evidence. | *Can I build a distributed system that stays correct under concurrency and partial failure, and prove it?* | Yes. Complete, demonstrable, and interview-ready on its own — and real AWS experience is banked at A4. |
| **B — Production delivery** | CI/CD with GitHub Actions, full AWS deployment on ECS Fargate, one bounded EKS exercise. | *Can I ship and operate that system the way a team does?* | Yes. Deploys the Phase A artifact; adds no domain behavior. |

Phase B is a second project that happens to deploy the first. Treating the two as one is the primary risk to completion identified in this revision.

Phase A is the thesis. If only Phase A is ever finished, the project has succeeded at its stated goal.

## Why auctions create the right pressure

Unchanged from 1.0, and still the strongest reason to keep this domain.

An auction concentrates writes on one authoritative record. Concurrent bids must be evaluated against status, time, minimum increment, bidder rules, and prior idempotency keys while preserving one winner and current price. That produces hot database records and, because auction facts are partitioned by `auctionId`, potentially hot Kafka partitions. The correct concurrency mechanism (optimistic, pessimistic, or an atomic database operation) must be selected by measurement and proven with invariant tests rather than promised in advance.

Downstream work broadens the failure surface. Facts may be duplicated or redelivered. Publication must survive a process crash after a local commit. Search indexes, notification views, and live streams may be eventually consistent and visibly lag. Retries can amplify load; poison messages need quarantine; replay must rebuild projections without repeating emails or charges. Auction close, order creation, payment, and notification delivery span independent owners, so partial success and recovery are normal conditions to design for.

The central distinction is consequence: search results, notification views, and live streams may lag, but accepting an invalid bid or charging twice may not. Authoritative owners enforce hard invariants synchronously in their local transaction; asynchronous consumers expose lag and recover safely.

## Bounded ambition

Unchanged. The first useful marketplace supports identity, marketplace profiles, classified listing drafts and images, auction publication and lifecycle, concurrent bidding, live updates, post-auction orders, provider-backed payments, notification delivery, and durable history. Initial discovery uses ordinary browse and filters.

**Staged:** nothing. The stack reduction (ADR-035 through ADR-039) removed the staged tier entirely — items are either in the baseline or excluded, since `staged` had become a place where unjustifiable tooling waited rather than a genuine later increment.

**Open:** payment provider, notification delivery provider, frontend libraries, schema registry, public license. See [05](05-exclusions-and-open-questions.md).

**Excluded from the baseline:** Spring Cloud Stream, Eureka, AWS API Gateway, WebSockets by default, CRUD microservices with no independent responsibility, and — new in 2.0 — Nx as build orchestration while the repository has no frontend.

## Definition of success

BidPoint succeeds when a later implementation can demonstrate the documented invariants under concurrency and failure, explain every ownership boundary, trace a request or event across components, and recover from redelivery or outage without corrupting authoritative state or repeating externally visible effects.

**2.0 makes the artifact explicit.** Success is evidenced by things a reader or interviewer can be shown: a concurrency test proving one price and one winner under contention, a measured hot-partition result, a crash-window test across the outbox boundary, a reconciliation trace after an ambiguous provider outcome, and load figures with the conditions under which they were taken. The acceptance-evidence column in [03 Delivery roadmap](03-delivery-roadmap.md) is the deliverable, not a checklist alongside it.

More technologies, deployables, or business rules do not improve that outcome unless they supply missing evidence. Neither does more documentation: 1.0's design corpus is treated as complete, and 2.0 deliberately adds six documents rather than twenty-four.
