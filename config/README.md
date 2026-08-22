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
- Environment-specific application settings

## What does not belong here

- **Plaintext secrets — never.** Passwords, tokens, API keys, and connection strings containing credentials belong in a dedicated secret-management system such as AWS Secrets Manager or HashiCorp Vault. This directory is in Git; anything committed here is permanent and readable.
- Kubernetes workload definitions — see [`gitops/`](../gitops/README.md)
- Infrastructure definitions — see [`infra/`](../infra/README.md)
- Application code

## Interacts with

- [`backend/`](../backend/README.md) and [`frontend/`](../frontend/README.md) — the applications that consume these settings
- [`gitops/`](../gitops/README.md) — references configuration into running workloads; the two must not both own the same value

## Likely later

Per-environment configuration sets. If BidPoint adopts a centralized configuration service such as Spring Cloud Config, this directory can serve as its backing source.
