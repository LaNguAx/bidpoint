# Product domain and capabilities

Status: Superseded by [design-system 2.0](../2.0/README.md)
Last validated: 2026-08-01

This document defines the target BidPoint marketplace behavior. It is a product and domain contract for future implementation, not a claim that these capabilities exist today.

## Actors and identity

A visitor can browse published listings. An authenticated marketplace member can maintain a profile, sell, bid, buy, and inspect durable history subject to object-level authorization. An operator can perform explicitly authorized support or audit actions. External payment and notification providers are untrusted integrations at the platform boundary.

Registration, authentication, passwords, MFA, sessions, and tokens belong to Keycloak. A marketplace profile is separate business data keyed by the validated Keycloak subject. Profile onboarding verifies trusted Keycloak identity evidence and activates the marketplace profile idempotently: repeating onboarding for the same subject returns the existing activation rather than creating another profile. The producer-owned `profiles` fact `UserRegistered` means that this marketplace profile activation committed. It is not the Keycloak credential-creation event. Profile requests never accept a body-supplied identity when the subject can be derived from the token.

## Capability map

| Capability | Expected behavior | Authority |
| --- | --- | --- |
| Register and authenticate | Establish an identity and obtain OIDC/OAuth2 tokens. | Keycloak |
| Maintain marketplace profile | Store display and marketplace data separately from credentials. | `core-platform` / `profiles` |
| Prepare a listing | Create and edit a draft, upload image objects, retain image metadata, validate publish readiness, and publish. | `core-platform` / `listings`; S3 stores objects |
| Classify and discover | Assign categories and classification rules; browse and filter initially. | `core-platform` / `catalog` and `listings` |
| Full-text search | Index published facts and expose REST queries. | **Staged:** `search-service` with OpenSearch |
| Run an auction | Schedule, open, close, cancel, settle, and initiate close behavior. | `core-platform` / `auctions` |
| Accept bids | Enforce status, minimum increment, bidder rules, idempotency, concurrency, and one current price/winner. | `bidding-service` |
| Stream live state | Deliver bounded-replay auction and bid updates over SSE; require refetch after a replay gap. | `realtime-service` |
| Preserve trade history | Create a post-auction order and durable buyer/seller views of it. | `core-platform` / `orders` |
| Collect payment | Integrate with a provider, verify webhooks, retry safely, and retain a payment audit trail. | `payment-service` |
| Notify members | Derive notification intent from business facts and perform delivery as idempotent work. | `notification-service` and `notification-worker` |
| Audit and recover | Retain authoritative transitions, stable identifiers, attempts, and recovery evidence. | Each owner for its state |

## Lean lifecycle model

The lifecycle vocabulary is intentionally small. Detailed pricing policies, dispute handling, refunds, shipping, tax, and reputation are outside the current domain unless introduced by a later decision.

### Listing

`DRAFT -> PUBLISHED`

- Only the owner may edit a draft.
- Publish requires required content, valid category classification, usable image metadata, and auction configuration to be ready.
- Published facts may feed eventually consistent projections; the authoritative listing remains in `listings`.

### Auction

`SCHEDULED -> OPEN -> CLOSING -> CLOSED -> SETTLED`

`SCHEDULED` or `OPEN` may enter `CLOSING` for cancellation and reach `CANCELLED` only under an explicitly authorized policy. Terminal transitions are audited.

- Core `auctions` owns the authoritative `closeAt` and a monotonically increasing `lifecycleVersion`. The `bidding-service` projection retains both. A stale opening projection may conservatively reject a valid bid, but lifecycle lag must never permit a bid after close or cancellation has been fenced.
- An auction cannot accept a bid unless the projection says it is open and trusted server/database time is strictly before `closeAt`; client-supplied time is ignored.
- Close or cancel moves Core to `CLOSING`, then Core sends an idempotent Bidding REST fence/finalize command. Its stable acknowledgement/final bid outcome is not an order trigger. Core retries the same identity after ambiguity. After close acknowledgement, `auctions` atomically commits `CLOSED` and one producer-owned `AuctionClosed` through the durable Spring Modulith event-publication/outbox mechanism; cancellation commits `CANCELLED` without an order trigger.
- `AuctionClosed` carries winner/no-winner, final price, lifecycle version, and stable identity. Reserve handling may yield no winner; it never yields multiple winners.
- `auctions` owns timing and lifecycle. It does not become the source of truth for bids, current price, or winner.
- Settlement follows successful creation of the post-auction trade and the required payment outcome; it does not imply that `orders` charges money.

### Bid

A bid is accepted or rejected; accepted bids are immutable facts.

- Reusing the same idempotency key for the same logical request returns the recorded result and cannot create another accepted bid.
- A bid is accepted only while the local auction-state projection is eligible, its amount satisfies the minimum increment, and the bidder satisfies the lightweight bidder rules.
- For each auction there is exactly one authoritative current price and at most one current winner after every committed acceptance.
- Competing requests must be serialized or conflict-detected at the authoritative state boundary. The exact database concurrency technique remains an implementation measurement decision.

### Order and payment

An order is created only from auctions-owned `AuctionClosed` after the `CLOSED` commit. `orders` deduplicates by event/auction identity. A bidding-owned final-outcome fact is audit/projection input only and cannot create an order. Payment has its own lifecycle in `payment-service`; no retry can produce a second charge for the same payment intent.

## Consistency promises

| Experience | Consistency expectation | Client/recovery behavior |
| --- | --- | --- |
| Bid acceptance and current winner/price | Authoritative and immediately invariant-preserving | Reject conflicts or invalid bids; retry only with the same idempotency key |
| Payment charge and webhook application | Authoritative and idempotent | Reconcile provider state and deduplicate webhook/provider identifiers |
| Listing and auction commands | Local transactional consistency at the owning module | Return the owner's committed result |
| Order creation from auction close | Triggered only by auctions-owned `AuctionClosed` after `CLOSED`; one logical order | Deduplicate by event/auction identity and expose processing state |
| Search results | **Staged**, eventually consistent | Expose index freshness; allow authoritative refetch |
| Notification views and deliveries | Eventually consistent | Show pending/failure state where useful; retry boundedly |
| SSE live updates | Eventually consistent with bounded replay | Reconnect with last event ID; refetch after a replay gap |

## Failure-aware product behavior

Commands return stable identifiers and correlation information so an ambiguous client timeout can be resolved without issuing a new logical operation. Asynchronous progress is observable rather than disguised as synchronous success. Users may see that an order, payment, or notification is processing. Operations retain enough audit data to distinguish a rejected command, a committed command awaiting publication, consumer lag, a retryable provider outage, and a quarantined poison message.
