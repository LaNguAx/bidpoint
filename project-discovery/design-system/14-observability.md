# Observability

Status: Canonical
Last validated: 2026-08-01

Observability makes the target platform's partial failures, lag, concurrency pressure, and recovery visible. It does not turn every signal into the same thing: metrics measure aggregates over time, traces follow a request/cause path, and logs preserve discrete redacted diagnostic records. This is target instrumentation and operations design, not evidence of running dashboards or alerting.

## Application signal contract

Applications emit Micrometer metrics; use Micrometer Observation-based instrumentation through an OpenTelemetry bridge/exporter for traces; and write structured JSON logs to stdout. Logs carry correlation and trace identifiers where available, use stable event/job/operation identifiers when relevant, and redact bearer tokens, passwords, credentials, private keys, card/payment data, raw authorization headers, and unnecessary personal data.

Every public REST request receives or propagates a correlation ID and W3C trace context. Internal REST passes them onward. Kafka envelopes retain correlation ID, causation ID, event ID, producer, and trace context where appropriate; RabbitMQ jobs retain stable job/intent IDs plus correlation/causation metadata. A consumer creates a new local processing span linked/continued from the message context and logs only safe identifiers. Background relays, scheduler work, retry, DLQ, and reconciliation flows also preserve causation. This supports diagnosis without pretending a trace proves an authoritative commit or exposing a secret.

## Collection and destination ownership

| Environment | Metrics | Logs | Traces | Exploration |
| --- | --- | --- | --- | --- |
| Local `core` | Basic telemetry endpoint/path sufficient to validate signal wiring. | Structured redacted stdout. | Basic Collector path. | Limited development inspection. |
| Local `platform` | OpenTelemetry Collector Contrib 0.157.0 -> Prometheus 3.13.2. | OpenTelemetry Collector -> Loki 3.7.4; no Promtail. | OpenTelemetry Collector -> Tempo 3.0.2. | Grafana 13.1.1. |
| AWS target | ADOT Collector 0.48.0 -> Amazon Managed Service for Prometheus. | ADOT/agent collection -> CloudWatch Logs. | ADOT/OpenTelemetry Collector -> AWS X-Ray. | Amazon Managed Grafana. |

The legacy AWS X-Ray SDK/daemon is **Excluded**. Prometheus/AMP hold metrics and do not replace traces; Tempo/X-Ray hold tracing data and do not become metric backends; Loki/CloudWatch Logs hold redacted logs and do not become an event source of truth. Collector pipelines must have bounded queues/backpressure and clear failure behavior so telemetry outage does not compromise bid, payment, or notification correctness.

## Health semantics

| Signal | Meaning | Unsafe interpretation to avoid |
| --- | --- | --- |
| Liveness | The process can make progress and should not be restarted only for an ordinary downstream outage. | A liveness success does not mean it can safely serve business traffic. |
| Readiness | The instance can accept its assigned traffic safely: initialization/migrations and required owned dependencies are ready, or the endpoint is deliberately isolated from that dependency. | A running container or open port is not readiness. |
| Health detail | Operator-facing dependency/degradation information, redacted and access-controlled. | It must not disclose secrets, internal topology, or authorization detail. |

Readiness gates traffic and rollout advancement. A Kafka consumer, outbox relay, or worker can be degraded without falsely claiming completed downstream effects; its lag/backlog and retry/DLQ signals communicate the condition. Health checks never issue writes, drain queues, or invoke payment/notification providers.

## Service indicators, objectives, and alerts

Initial objectives and thresholds are set from baseline/load evidence, not invented universally in this document. Each SLI has an owner, label cardinality policy, dashboard, alert route, and runbook before paging is enabled.

| Area | Required SLI/signal | Alert or investigation condition |
| --- | --- | --- |
| HTTP RED | request rate, error ratio/class, latency distribution for `api-gateway` and each owner endpoint. | Sustained error/latency budget burn, split by route/status without user-ID cardinality. |
| JVM/runtime | Heap, GC pauses, threads, CPU, memory, restart/readiness state. | Saturation, crash/restart loop, or performance degradation correlated with HTTP/broker signals. |
| PostgreSQL | Connection-pool saturation, query/transaction latency, errors, locks, migration state. | Pool exhaustion, persistent latency/error, lock contention, backup/restore failure. |
| Outbox | Unpublished row count, oldest row age, relay failures/retries. | Age/backlog breaches a recovery objective; not merely a transient retry. |
| Kafka | Consumer lag, processing error/retry, partition skew/hot auction partition, DLQ/quarantine count. | Lag/age threatens projection freshness or retries/DLQ grow unexpectedly. |
| RabbitMQ | Queue depth, oldest message age, unacked/retry/DLQ/quarantine count, worker throughput. | Work obligation exceeds delivery objective, consumer loss, or unknown terminal state needs operator action. |
| Bidding | Accepted/rejected/conflict outcomes, idempotency replays, bid latency, fence/finalize failures. | Abnormal rejection/conflict, invariant/fence failure, or hot-auction saturation. |
| Payments | Webhook verification failures, reconciliation/UNKNOWN attempts, initiation/settlement errors. | Signature/replay anomalies or unresolved payment attempts; never emit payment data. |
| Notifications | Intent backlog, provider failure class, delivery latency, UNKNOWN/DLQ count. | Retry budget exhaustion or growing quarantined obligations. |
| SSE/realtime | Connections, reconnects, events delivered, replay-gap/refetch rate, Kafka-to-stream age. | Capacity exhaustion or sustained replay gaps/lag that degrade live experience. |
| GitOps/release | Argo CD sync/health, Argo Rollouts phase/abort, ready replica and rollout analysis state. | Drift, unhealthy sync, stalled/failed rollout, or rollback action. |

Bid acceptance, payment state, and other authoritative records retain their own audit evidence; monitoring is not the authority. Synthetic probes and alert tests must use safe, non-secret paths and cannot simulate a charge or production notification delivery by default.

## Operations and retention

Dashboards correlate edge request, owner API, database/outbox, broker consumer, and downstream delivery/realtime effects through safe identifiers. Runbooks define how to inspect lag, replay safely, reconcile an `UNKNOWN` provider attempt, roll back/forward a GitOps release, and preserve evidence. Retention, access, and cost policies are environment decisions: logs/traces require least-privilege operator access and redaction; high-cardinality labels and unbounded payload logging are prohibited.

**Staged:** full local platform dashboards/alerts, Istio telemetry/fault injection, advanced rollout analysis, chaos signals, and SLO thresholds set from meaningful baselines. **Excluded:** Promtail, legacy X-Ray SDK/daemon, secret-bearing logs, and the claim that metrics, traces, or logs alone supply end-to-end exactly-once proof.
