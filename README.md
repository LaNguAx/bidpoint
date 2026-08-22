# BidPoint

An event-driven auction platform engineered to production standards across architecture, infrastructure, and operations.

BidPoint is an **organization-level monorepo**. It holds the application source, runtime configuration, deployment state, cloud infrastructure, and automation for the whole system — the complete lifecycle of a production distributed application in one repository.

The system is in early scaffolding. This documentation establishes the architectural thesis and module boundaries before implementation begins.

## Modules

| Path | Owns |
| --- | --- |
| [`backend/`](backend/README.md) | Server-side application systems |
| [`frontend/`](frontend/README.md) | User-facing clients — web and mobile |
| [`config/`](config/README.md) | Application runtime configuration |
| [`gitops/`](gitops/README.md) | Desired state of Kubernetes workloads |
| [`infra/`](infra/README.md) | Cloud and platform infrastructure |
| [`.github/`](.github/AUTOMATION.md) | CI/CD and repository automation |
| [`docs/`](docs/README.md) | Architecture and operational knowledge |

## Boundaries

Six concerns, one owner each. A change belongs to exactly one of them.

```text
SOURCE CODE           backend/  frontend/   what the system does
APPLICATION CONFIG    config/               how an application behaves
DEPLOYMENT STATE      gitops/               how and where it runs on Kubernetes
CLOUD INFRASTRUCTURE  infra/                what underlying infrastructure exists
AUTOMATION            .github/              how changes are validated and delivered
KNOWLEDGE             docs/                 why the system is shaped this way
```

The distinction that matters most:

```text
config/  → how an application behaves
gitops/  → how and where an application runs in Kubernetes
infra/   → what underlying infrastructure exists
```

## External systems

These are runtime systems BidPoint depends on. They may be **provisioned or configured** from this repository, but their runtime state and data never live in Git.

| System | Role |
| --- | --- |
| ECR | Built container artifacts |
| S3 / CDN | Built frontend artifacts |
| AWS Secrets Manager / Vault | Secrets |
| EKS | Workload runtime |
| RDS / databases | Persistent data |
| Kafka / RabbitMQ | Messaging infrastructure |
| Observability platforms | Runtime telemetry |

## Working in this repository

Read [`CLAUDE.md`](CLAUDE.md) before making changes. Read the module's own README before changing a module. Record significant architectural decisions under `docs/adr/`.
