# Thesis

## What BidPoint is

A real-time auction marketplace. Users list items, auctions open and close on a schedule, people bid concurrently, the highest bid wins, an order is created, payment is taken, notifications go out, and everything updates live in the browser.

The domain is deliberately small. It is not trying to be a feature-complete commerce platform, and feature breadth is not the point.

## Why it's being built

**To learn the practical, industry-standard engineering that senior backend engineers actually do** — by building a system where the hard parts are unavoidable, rather than reading about them.

Two goals, and they mostly agree:

1. **Employability.** Spring/Java backend roles ask for distributed systems, Kubernetes, AWS, Kafka, and testing discipline. This project produces demonstrable experience with all of them.
2. **Genuine understanding.** Working as a backend developer without a practical grasp of how distributed systems fail is a real gap. Knowing the vocabulary is not the same as having watched a system lose a message and recover.

This is a **learning project, not a delivery project**. A feature that works but isn't understood is a failed step. If the choice is between shipping the next stage and understanding the current one, understanding wins.

## Why auctions specifically

Most side projects pick a CRUD domain and bolt on Kafka for the résumé. Nothing forces correctness, so nothing is learned.

An auction is different. It concentrates concurrent writes on **one record** with a hard invariant: exactly one current price, at most one current winner. Two hundred people bidding in the final second is a real race, not a hypothetical one. You cannot fake your way past it, and the mechanism you choose — optimistic locking, pessimistic locking, or an atomic database operation — has to be picked by measurement and proven by test.

From that one pressure point, everything else follows naturally:

- **Hot records** in the database, and because auction events are partitioned by auction, **hot partitions** in Kafka.
- **Idempotency**, because a client that times out will retry and must not bid twice.
- **The outbox problem**, because committing a bid and publishing the event can't be one atomic action across two systems.
- **At-least-once delivery**, duplicates, and consumer deduplication.
- **Partial failure**, because auction close, order creation, payment, and notification span independent owners — some will succeed while others fail.
- **Eventual consistency with visible lag**, in search, notifications, and the live feed.
- **Recovery**: replay, poison messages, dead-letter queues, and reconciliation after an ambiguous external call.

## The line that matters

**Some things may lag. Some things may never be wrong.**

Live updates, notification views, and search results are allowed to be a few seconds stale — that's the nature of derived, asynchronously-built state.

Accepting an invalid bid or charging a card twice is never acceptable, and "eventual consistency" is not an excuse for either.

Authoritative owners enforce hard invariants **synchronously, inside their own database transaction**. Asynchronous consumers expose their lag and recover safely. Knowing which category a given piece of state belongs in is most of the skill.

## What "done" looks like

Not "the feature works." The deliverable is **evidence**:

- a test that spawns concurrent bidders and proves exactly one price and one winner survived;
- a test that kills the process between the database commit and the event publish, and shows nothing was lost;
- a measurement of what actually happens to a hot partition under load, with numbers;
- a trace showing a duplicate event arriving and being correctly ignored;
- a reconciliation after a payment provider times out ambiguously, proving no double charge.

These are also, not coincidentally, the answers to the interview questions that separate candidates. "How would you guarantee one winner under concurrency?" is a much better answer when it ends with *"— here's the test, and here's what the measurement showed."*

## Scope discipline

Every tool and every service must map to one of the lessons above. If it can't, it gets removed — not deferred, not marked "later." That rule is what kept the stack at roughly twenty tools instead of forty. See [03 Tech stack](03-tech-stack.md).

More technology does not improve the outcome. More evidence does.
