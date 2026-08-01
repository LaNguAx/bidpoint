# Roadmap

Twelve stages in two phases. Each produces something that runs and something that proves it works.

**A stage is done when the evidence exists, not when the feature works.** That's from [01 Thesis](01-thesis.md) and it's the rule that makes this a learning project rather than a delivery project.

## Ordering logic

Three deliberate choices:

1. **Bidding concurrency is A3, not the end.** It's the thesis. Everything else exists to surround it.
2. **AWS is A4, not stage eleven.** Deploy something real early, while the system is small enough to understand, so cloud experience is banked rather than held hostage to finishing everything. Same Kubernetes on both sides, so nothing is thrown away.
3. **Kubernetes from A1.** Learning k8s against the gateway and one backend is far easier than retrofitting it onto the finished system. There's no Compose phase and no migration.

---

## Phase A — Correctness

Runs on local k3d. Tests use Testcontainers and need no cluster.

### A1 — Foundation
Gradle multi-project build via wrapper, Amazon Corretto 25, Spring Boot 4.0.x on the Spring Cloud 2025.1.x release train, `com.bidpoint` namespace, module boundaries, and quality gates. A k3d cluster with PostgreSQL via **CloudNativePG**. Spring Cloud Gateway and one backend deployed and reachable through Traefik's Gateway API provider.

**Done when:** `./gradlew check` passes; `ApplicationModules.verify()` **fails** when you deliberately violate a boundary; Spotless and JaCoCo are wired; `kubectl get pods` shows a healthy gateway, backend, and CloudNativePG cluster; an `HTTPRoute` reaches the backend through Spring Cloud Gateway; a Flyway migration has run; you can explain what the operator's reconciliation loop is doing.

### A2 — Core domain and identity
Profiles, catalog, listings, auction scheduling and lifecycle. Keycloak, JWT validation, object-level authorization.

**Done when:** module ownership tests pass; an invalid token is rejected by the resource server; a non-owner is blocked from editing a listing, with a test; no module writes another's tables.

### A3 — Bidding, concurrency, idempotency ← **the thesis**
Safe concurrent bids, the fence/finalize handshake, hot-record authority.

**Done when:** a concurrency test with many simultaneous bidders proves **exactly one price and one winner** survived; reusing an idempotency key returns the recorded result instead of creating a second bid; a fence retry after simulated ambiguity produces the same outcome; and the concurrency mechanism was chosen **by measurement**, with the numbers written down.

### A4 — AWS tracer bullet
Create the persistent `bootstrap/` and ephemeral `environment/` Terraform stacks, stand up **EKS**, and run the gateway plus **one** backend on it. Exercise IAM, Pod Identity, ECR, RDS, External Secrets Operator with SSM Parameter Store, Gateway API through the AWS Load Balancer Controller, external-dns, ACM, and selected EKS control-plane logs in CloudWatch.

**Done when:** `bootstrap/` holds the state bucket, ECR repositories, Route53 zone, ACM certificate, GitHub OIDC role, budget alarm, and secret backends; `environment/` goes from nothing to a reachable HTTPS service on EKS; the image is pulled from ECR by digest; the workload uses Pod Identity; its secret comes through External Secrets Operator and is absent from Git, the image, and Terraform state; an `HTTPRoute` is served by the ALB; selected control-plane logs appear in CloudWatch; destroying `environment/` removes the EKS, node, ALB, and RDS charges while `bootstrap/` deliberately persists; the session cost is recorded against the current rate table in [03 Tech stack](03-tech-stack.md).

*Small on purpose. The goal is proving the whole path end to end, not deploying the finished system.*

### A5 — Kafka facts, outbox, replay
Durable facts, at-least-once delivery, visible lag.

**Done when:** state and outbox row commit atomically; a test that kills the process **between commit and publish** shows nothing is lost; a duplicate event is correctly ignored by consumer deduplication; per-`auctionId` ordering holds; a replay rebuilds a projection without re-sending anything externally visible.

### A6 — Realtime and SSE
Live updates, bounded replay, reconnect. Multiple `core-platform` replicas make fan-out a real problem.

**Done when:** SSE is served from more than one replica through Spring Cloud Gateway; a client reconnects through the gateway with its last event ID and gets the gap; a replay gap correctly triggers a REST refetch; proxying through the WebFlux gateway is covered by a test; SSE is demonstrably not command authority.

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

The flow: push → Actions runs the Gradle build and tests → Jib publishes the image to ECR by digest → Actions commits the digest to `deploy/dev/values.yaml` in the same repository → Argo CD reconciles the cluster. Promotion copies that digest to stage and then prod through pull requests.

**Done when:** the pipeline runs `./gradlew check` including Testcontainers; Jib publishes an `arm64` or multi-architecture image and deployment pins its digest, never `latest`; GitHub authenticates to ECR through IAM OIDC with no static AWS key; path filters plus `[skip ci]` prevent the digest commit from triggering a build loop; an ApplicationSet produces dev, stage, and prod Applications from one chart and three values files; promotion is a pull request; Argo CD shows the app synced and healthy; **a manual `kubectl edit` is automatically reverted**; rollback is `git revert`; CI has no cluster credentials and never deploys directly.

### B2 — Full AWS deployment
All four domain deployables plus `api-gateway` on EKS across **dev, stage, and prod namespaces**, with the **same operators, Helm charts, and `HTTPRoute` resources used locally**. Each namespace has its own Strimzi Kafka, RabbitMQ cluster, Keycloak, Redis, and RDS instance; all three environments run simultaneously. One shared ALB serves the three hostnames.

**Done when:** every runtime deployable is healthy behind the shared ALB; the same manifests that run on k3d run here, with only values and `GatewayClass` differing; workload identity is Pod Identity, not static keys or IRSA; External Secrets Operator reads dev and stage values from SSM Parameter Store and prod values from Secrets Manager; Loki and Tempo retain evidence in S3 across teardown; a full `environment/` create-and-destroy cycle completes while `bootstrap/` remains; **cost per session is recorded and compared with the ~$0.54/hr On-Demand and ~$0.33/hr Spot estimates**.

### B3 — Resilience and scale
HPA under load. Chaos. Whatever the measurements from A9 said was weakest.

**Done when:** HPA scales a service under k6 load and scales back down; a deliberately killed pod is replaced without dropping work; a broker restart is survived without losing a message; results are written down with what surprised you.

---

## Rules that hold at every stage

- Gradle via the checked-in wrapper is the only build authority.
- Owners write only their own data. No shared business models.
- No secrets in Git, Terraform state, Helm values, events, jobs, or logs.
- Ordinary tests make no external network calls.
- The ephemeral AWS `environment/` stack is destroyed after every session; the low-cost `bootstrap/` stack persists by design.
- Coverage is a diagnostic signal, never a target to hit.
- Every tool maps to a lesson, or it doesn't go in.
