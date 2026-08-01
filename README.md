# BidPoint

A real-time auction marketplace, built to learn the practical, industry-standard engineering that senior backend engineers actually do.

Java · Spring Boot · PostgreSQL · Kafka · RabbitMQ · Redis · Kubernetes · AWS

## Why

Auctions concentrate concurrent writes on one record with a hard invariant — exactly one current price, at most one winner. That single pressure point forces everything worth learning: idempotency, transactional outboxes, at-least-once delivery, hot partitions, partial failure, and recovery. You can't fake your way past a race condition.

It's a learning project. The deliverable is evidence that the system stays correct under concurrency and failure, not a feature list.

## Docs

| | |
| --- | --- |
| [Thesis](docs/01-thesis.md) | What this is and why it's being built. |
| [Architecture](docs/02-architecture.md) | How it's shaped, and the rules that must never break. |
| [Tech stack](docs/03-tech-stack.md) | The tools, why each earns its place, and the local and AWS environments. |
| [Roadmap](docs/04-roadmap.md) | What gets built next, and what proves it's done. |

## Status

**Nothing is implemented yet.** No build, no dependencies, no infrastructure. Next step is A1 in [the roadmap](docs/04-roadmap.md).

[`old/`](old/) holds superseded design material, kept for history only.
