# BidPoint — A Distributed Systems Reference Platform

Status: Canonical
Last validated: 2026-08-01

## Thesis

BidPoint is a deliberately bounded auction marketplace used to practise production-grade backend engineering and distributed-systems behavior. The marketplace is credible enough to make correctness matter, yet intentionally avoids domain complexity that would hide the engineering lessons. This repository is documentation-only: the architecture below is a target design, not an account of software that already exists.

The platform is for engineers who want hands-on experience designing, implementing, testing, operating, and recovering a distributed backend. It is also for reviewers who need an explicit contract against which later implementation choices can be challenged. It is not optimized as an interview exercise or as a feature-complete commercial marketplace.

## Learning outcomes

The target design creates concrete opportunities to learn:

- service and module ownership, with behavior exposed through APIs and events rather than shared tables;
- local ACID transactions combined with transactional outboxes and at-least-once delivery;
- concurrency, ordering, and idempotency where many users bid on the same auction;
- appropriate use of REST, SSE, Kafka, and RabbitMQ according to message purpose;
- derived projections, observable lag, poison-message handling, replay, and partial-failure recovery;
- OAuth2/OIDC identity, independently enforced resource-server security, object-level authorization, and secret isolation;
- production concerns such as hot records, hot Kafka partitions, bounded retries, audit trails, load tests, and failure injection.

Every technology and deployable must teach one of these responsibilities. A split that exists only to make the system look distributed is a design failure.

## Bounded ambition

The first useful marketplace supports identity, marketplace profiles, classified listing drafts and images, auction publication and lifecycle, concurrent bidding, live updates, post-auction orders, provider-backed payments, notification delivery, and durable history. Initial discovery uses ordinary browse and filters.

**Staged:** `search-service` and OpenSearch add full-text search after the core marketplace is sound. Istio ambient service identity, authorization policies, mTLS, and fault injection are also staged.

**Open:** the payment provider, notification delivery provider, frontend libraries, Kafka/MSK operating mode, schema registry, and public license. These documents deliberately do not choose them.

**Excluded from the baseline:** Spring Cloud Stream, Eureka, AWS API Gateway, WebSockets by default, and CRUD microservices with no independent consistency, scaling, or failure responsibility.

## Why auctions create the right pressure

An auction concentrates writes on one authoritative record. Concurrent bids must be evaluated against status, time, minimum increment, bidder rules, and prior idempotency keys while preserving one winner and current price. That produces hot database records and, because auction facts are partitioned by `auctionId`, potentially hot Kafka partitions. The correct concurrency mechanism (optimistic, pessimistic, or an atomic database operation) must be selected by measurement and proven with invariant tests rather than promised in advance.

Downstream work broadens the failure surface. Facts may be duplicated or redelivered. Publication must survive a process crash after a local commit. Search indexes, notification views, and live streams may be eventually consistent and visibly lag. Retries can amplify load; poison messages need quarantine; replay must rebuild projections without repeating emails or charges. Auction close, order creation, payment, and notification delivery span independent owners, so partial success and recovery are normal conditions to design for.

The central distinction is consequence: search results, notification views, and live streams may lag, but accepting an invalid bid or charging twice may not. Authoritative owners enforce hard invariants synchronously in their local transaction; asynchronous consumers expose lag and recover safely.

## Definition of success

BidPoint succeeds when a later implementation can demonstrate the documented invariants under concurrency and failure, explain every ownership boundary, trace a request or event across components, and recover from redelivery or outage without corrupting authoritative state or repeating externally visible effects. More technologies, deployables, or business rules do not improve that outcome unless they supply missing evidence.
