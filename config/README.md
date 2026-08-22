# config/

Application-level runtime configuration: the settings that determine **how an application behaves**.

This is distinct from deployment configuration and from infrastructure:

```text
config/  → how an application behaves
gitops/  → how and where an application runs in Kubernetes
infra/   → what underlying infrastructure exists
```

## What belongs here

- Service endpoints
- Kafka and RabbitMQ endpoints
- Feature flags
- Timeouts and retry settings
- Application behavior switches
- Logging configuration
- Per-environment values for any of the above

## What does not belong here

- **Plaintext secrets — never.** Passwords, tokens, API keys, and connection strings containing credentials belong in a dedicated secret-management system such as AWS Secrets Manager or HashiCorp Vault. This directory is in Git; anything committed here is permanent and readable.
- Kubernetes workload definitions — see [`gitops/`](../gitops/README.md)
- Infrastructure definitions — see [`infra/`](../infra/README.md)
- Application code

## The boundary with gitops/

Both directories hold settings, and both can end up as environment variables inside a running container. The distinction is not the format — it is **which process reads the value**.

| | `config/` | `gitops/` |
| --- | --- | --- |
| Read by | the application | Kubernetes |
| Read when | at startup, and on refresh | at pod creation |
| Example | bid extension window, retry count, log level | replicas, memory limit, probe path |

Two tests when a setting is ambiguous:

1. **Generic or domain-specific?** Every Deployment on earth has `replicas` — that is `gitops/`. Only BidPoint has a bid extension window — that is `config/`.
2. **Would it survive deleting the application code?** `replicas: 3` still means something. `bid.timeout: 30s` does not; it exists only because a service reads it.

A workload in [`gitops/`](../gitops/README.md) may set **only** these environment variables:

```text
SPRING_PROFILES_ACTIVE     which environment the service is running as
SPRING_APPLICATION_NAME    which configuration set to request
SPRING_CONFIG_IMPORT       where the configuration server is
+ secret-backed variables sourced from Kubernetes Secrets
```

Those are bootstrap and identity: a service cannot ask the configuration server for its settings until it knows where the server is and which environment it belongs to. Everything else it reads as behavior belongs here.

Any other environment variable in a `gitops/` manifest means a property is declared in two places. In Spring Boot this fails silently — relaxed binding makes `BID_TIMEOUT` and `bid.timeout` the same property under two spellings, and an environment variable can outrank a value served from the configuration server. The file here then looks correct and is ignored.

## Delivery

Services read these settings from a configuration server rather than from mounted files, so a value can change without rebuilding or redeploying the workload that consumes it. The backend is Spring Boot, which makes Spring Cloud Config the expected implementation.

The configuration server is itself a workload: [`gitops/`](../gitops/README.md) deploys it, and this directory is what it serves. None of this is implemented — no configuration sets exist and no server is deployed.

## Interacts with

- [`backend/`](../backend/README.md) and [`frontend/`](../frontend/README.md) — the applications that consume these settings
- [`gitops/`](../gitops/README.md) — deploys the configuration server and the workloads that read from it; the two must not both own the same value

## Likely later

Per-environment configuration sets, and the configuration server's own deployment under `gitops/`.
