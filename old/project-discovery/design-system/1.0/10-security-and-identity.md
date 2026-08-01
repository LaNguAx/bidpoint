# Security and identity

Status: Superseded by [design-system 2.0](../2.0/README.md)
Last validated: 2026-08-01

This document defines BidPoint's target security boundary. It applies to a future implementation and is not evidence that identities, credentials, infrastructure policies, or provider integrations exist in this documentation-only repository.

## Identity, authentication, and authorization

Keycloak owns identities, passwords, MFA, sessions/tokens, and OIDC/OAuth2 flows. BidPoint stores a marketplace profile separately in Core `profiles`, keyed by the validated Keycloak subject; it never stores credentials or treats a body-supplied user ID as authoritative when the token can supply the subject. `profiles` activates onboarding idempotently after verified Keycloak identity evidence and then owns the `UserRegistered` application fact. That fact means marketplace profile activation, not Keycloak credential creation.

`api-gateway` performs early JWT rejection and coarse route concerns before forwarding public traffic. Every backend is independently a Spring Security OAuth2 Resource Server that validates JWTs; a gateway validation result is not transitive trust. The target service or module owns business authorization and object-level access decisions, such as whether the authenticated subject may edit a listing, bid on an auction, view an order, or perform an authorized operator action.

| Boundary | Required control | It must not be mistaken for |
| --- | --- | --- |
| Keycloak | Identity lifecycle, OIDC/OAuth2, passwords, MFA, sessions/tokens | Marketplace profile or application business authorization |
| `api-gateway` | Early token rejection, CORS, routing, rate limits, trace/correlation propagation | Final backend authorization |
| Owner backend | JWT validation plus business/object authorization | Trust in client-supplied subject or gateway-only enforcement |
| Provider callback | Verified signature and replay protection | An authenticated end-user request |
| Service-to-service | **Staged** Istio ambient identity/mTLS and policies | Replacement for JWT validation or authorization |

Keycloak has a distinct authentication host/path and is not routed through Spring Cloud Gateway. The public front door remains explicitly `Spring Cloud Gateway`, behind pinned Traefik locally or Route 53, AWS WAF, and ALB via AWS Load Balancer Controller in AWS. Kubernetes Gateway API is configuration, not the runtime gateway; AWS API Gateway is **Excluded**.

## Data and credential protection

Remote secrets are designed for AWS Secrets Manager with EKS Pod Identity and the Secrets Store CSI driver/provider. Application credentials, tokens, private keys, provider secrets, and connection credentials are never committed to source control, hard-coded in documentation examples, emitted to logs, or embedded in event/job payloads. Access is least privilege: a workload receives only the secret and cloud/database permissions necessary for its owned responsibility.

Payment card data never enters BidPoint scope. The payment provider is **Open** until selected. Once a provider is selected, `payment-service` verifies webhook signatures, applies replay protection using stable provider/event identities, keeps an audit trail, and reconciles ambiguous provider requests before retry. Validity of a webhook signature is a required ingress check, but payment state changes still enforce the service's idempotency and state-transition rules.

At-rest and in-transit controls are selected in the future implementation/deployment design; they do not weaken the data ownership rule. A service cannot gain read/write access to another owner's schema just because it can authenticate to the same PostgreSQL deployment. Redis remains acceleration/ephemeral coordination, and **Staged** OpenSearch remains a derived store with only the data/permissions justified by its search responsibility.

## Edge abuse controls and safe observability

The gateway owns CORS policy and Redis-backed rate limits for public routes. Route controls are designed per resource sensitivity: authentication, bid submission, payment initiation, and SSE connection establishment need abuse controls that avoid turning a noisy client into a platform-wide outage. They do not replace owner-side validation, idempotency, concurrency checks, or payment reconciliation.

Correlation IDs and trace context are propagated across REST and, where appropriate, event envelopes so an authorized operator can diagnose failures. Audit records preserve the who/what/when/result required for sensitive state transitions, payment/webhook handling, delivery attempts, and recovery. Logs are structured and redacted: no bearer tokens, passwords, secrets, card data, raw provider credentials, or unnecessary personal content. Error responses avoid exposing internal topology, authorization details, or confidential validation state.

## Threat-boundary notes

| Threat boundary | Target mitigation |
| --- | --- |
| Browser -> public API | TLS in deployment, gateway JWT rejection/CORS/rate limits, backend JWT and object authorization |
| Browser -> SSE | Authenticated connection, bounded replay, refetch on gap; no SSE event grants command authority |
| Service -> service | Owner APIs, independent JWT validation; **Staged** Istio ambient identity/mTLS/policies add defense in depth |
| Kafka/RabbitMQ consumers | Least-privilege broker credentials, stable-ID deduplication, payload minimization, audit and DLQ/quarantine operations |
| Payment provider -> webhook endpoint | Signature validation, replay protection, idempotent state handling, safe reconciliation |
| Notification provider | Selection requires stable-key idempotency and reconciliation/status lookup; ambiguous `UNKNOWN` attempts are quarantined without automatic replay; secret isolation and redacted attempt logs |
| Operator/support actions | Explicitly authorized scope, audited transitions, least privilege |

Event and job payloads contain stable identifiers and the minimum business data needed by the consumer. They must not carry access tokens, passwords, card data, or arbitrary unredacted request copies. DLQ/quarantine inspection follows the same access-control and redaction rules because failure records often contain sensitive context.

## Staged and excluded controls

**Staged:** Istio ambient service-to-service identity, mTLS, authorization policies, and fault injection. They complement application-layer resource-server validation and owner-level business authorization; they do not replace them.

**Open:** payment provider, notification delivery provider, and the remaining platform choices recorded elsewhere. **Excluded from the baseline:** AWS API Gateway, gateway-only authorization, committing application credentials, trusting identity from a request body, and accepting unsigned or replayed payment webhooks.
