# Roadmap

Twelve stages in two phases. Each produces something that runs and something that proves it works.

**A stage is done when the evidence exists, not when the feature works.** That's from [01 Thesis](01-thesis.md) and it's the rule that makes this a learning project rather than a delivery project.

## Ordering logic

Three deliberate choices:

1. **Bidding concurrency is A3, not the end.** It's the thesis. Everything else exists to surround it.
2. **AWS is A4, not stage eleven.** Deploy something real early, while the system is small enough to understand, so cloud experience is banked rather than held hostage to finishing everything. Same Kubernetes on both sides, so nothing is thrown away.
3. **Kubernetes from A1.** Learning k8s against a two-service system is far easier than retrofitting it onto nine. There's no Compose phase and no migration.

---

## Phase A — Correctness

Runs on local k3d. Tests use Testcontainers and need no cluster.

### A1 — Foundation
Maven reactor, `com.bidpoint` namespace, module boundaries, quality gates. A k3d cluster with PostgreSQL via **CloudNativePG**. One service deployed and reachable.

**Done when:** `./mvnw verify` passes; an ArchUnit or Modulith test **fails** when you deliberately violate a boundary; Spotless and JaCoCo are wired; `kubectl get pods` shows a healthy service; a Flyway migration has run against a CloudNativePG cluster; you can explain what the operator's reconciliation loop is doing.

### A2 — Core domain and identity
Profiles, catalog, listings, auction scheduling and lifecycle. Keycloak, JWT validation, object-level authorization.

**Done when:** module ownership tests pass; an invalid token is rejected by the resource server; a non-owner is blocked from editing a listing, with a test; no module writes another's tables.

### A3 — Bidding, concurrency, idempotency ← **the thesis**
Safe concurrent bids, the fence/finalize handshake, hot-record authority.

**Done when:** a concurrency test with many simultaneous bidders proves **exactly one price and one winner** survived; reusing an idempotency key returns the recorded result instead of creating a second bid; a fence retry after simulated ambiguity produces the same outcome; and the concurrency mechanism was chosen **by measurement**, with the numbers written down.

### A4 — AWS tracer bullet
Stand up **EKS** with Terraform and run **one** service on it. IAM, Pod Identity, ECR, RDS, Secrets Manager, ALB, CloudWatch.

**Done when:** `terraform apply` goes from nothing to a reachable service on EKS; the image was promoted to ECR by digest; secrets came from Secrets Manager and are not in the image; logs appear in CloudWatch; **`terraform destroy` returns to zero**; a billing alarm is set; the session cost is recorded and compared against the ~$0.21/hr estimate.

*Small on purpose. The goal is proving the whole path end to end, not deploying the finished system.*

### A5 — Kafka facts, outbox, replay
Durable facts, at-least-once delivery, visible lag.

**Done when:** state and outbox row commit atomically; a test that kills the process **between commit and publish** shows nothing is lost; a duplicate event is correctly ignored by consumer deduplication; per-`auctionId` ordering holds; a replay rebuilds a projection without re-sending anything externally visible.

### A6 — Realtime and SSE
Live updates, bounded replay, reconnect. Multiple `core-platform` replicas make fan-out a real problem.

**Done when:** SSE is served from more than one replica; a client reconnects with its last event ID and gets the gap; a replay gap correctly triggers a REST refetch; SSE is demonstrably not command authority.

### A7 — Kafka → RabbitMQ fan-out
Facts versus work. Policy separate from delivery. Competing consumers.

**Done when:** the full chain is observed end to end — Kafka fact → `notification-service` decision → durable intent and job outbox → RabbitMQ → **N workers competing**, each job executed exactly once; retries are classified; a poison job lands in the DLQ; an `UNKNOWN` provider outcome is reconciled before retry using a stable key.

### A8 — Orders and payment
One order per close. Provider adapter. Webhook recovery.

**Done when:** a redelivered `AuctionClosed` creates **exactly one** order; a simulated provider timeout is reconciled without double charging; webhook signature verification and replay protection are tested. A fake provider adapter is sufficient.

### A9 — Observability and load
The full LGTM stack: OpenTelemetry Collector, Prometheus, Loki, Tempo, Grafana. Failure injection. Capacity.

**Done when:** Grafana shows request rate, errors, latency, outbox age, consumer lag, and DLQ depth; logs are queryable across services in Loki; **one bid is visible as a single connected trace in Tempo** spanning REST → transaction → outbox → Kafka → notification-service → RabbitMQ → worker; Toxiproxy-induced failures show recovery; k6 results are recorded with their conditions; **hot-partition behavior is measured**, not assumed.

*That single end-to-end trace is the best artifact this project produces. Treat it as the deliverable.*

**Phase A is complete and demonstrable on its own.** If nothing further gets built, the goal is met — and A4 already banked real AWS experience.

---

## Phase B — Production delivery

No new domain behavior. This is about shipping and operating what already works.

### B1 — CI/CD with GitOps
GitHub Actions for CI, **Argo CD** for CD. Build strictly separated from deploy.

The flow: push → Actions builds and tests → image to ECR by digest → commit the digest to the GitOps repo → Argo CD reconciles the cluster.

**Done when:** the pipeline runs the full Maven verify including Testcontainers; images are tagged by digest, never `latest`; Argo CD shows the app synced and healthy; **a manual `kubectl edit` is automatically reverted** — proving reconciliation actually works; a rollback is a git revert, not a pipeline re-run; CI has no cluster credentials.

### B2 — Full AWS deployment
All four deployables on EKS, with the **same operators and Helm charts used locally** — Strimzi, RabbitMQ, Keycloak — against RDS and S3.

**Done when:** every service is healthy behind the ALB; the same manifests that run on k3d run here, with only values differing; workload identity is Pod Identity, not static keys; secrets come from Secrets Manager; a full create-and-destroy cycle completes; **cost per session is recorded**.

### B3 — Resilience and scale
HPA under load. Chaos. Whatever the measurements from A9 said was weakest.

**Done when:** HPA scales a service under k6 load and scales back down; a deliberately killed pod is replaced without dropping work; a broker restart is survived without losing a message; results are written down with what surprised you.

---

## Rules that hold at every stage

- Maven is the only build authority.
- Owners write only their own data. No shared business models.
- No secrets in Git, Terraform state, Helm values, events, jobs, or logs.
- Ordinary tests make no external network calls.
- AWS infrastructure is destroyed after every session.
- Coverage is a diagnostic signal, never a target to hit.
- Every tool maps to a lesson, or it doesn't go in.
