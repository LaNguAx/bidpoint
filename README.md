# BidPoint

An event-driven auction platform designed to production standards across architecture, infrastructure, and operations.

BidPoint is an **organization-level monorepo**. It holds the application source, runtime configuration, deployment state, cloud infrastructure, and automation for the whole system — the complete lifecycle of a production distributed application in one repository.

The system is in early scaffolding: this documentation establishes the architectural thesis and module boundaries; almost nothing is implemented yet.

## Modules

| Path | Owns |
| --- | --- |
| [`backend/`](backend/README.md) | Server-side application systems |
| [`frontend/`](frontend/README.md) | User-facing clients — `web/` and `mobile/` |
| [`config/`](config/README.md) | Backend application runtime configuration |
| [`gitops/`](gitops/README.md) | Desired state of Kubernetes workloads |
| [`infra/`](infra/README.md) | Cloud and platform infrastructure |
| [`.github/`](.github/AUTOMATION.md) | CI/CD and repository automation |
| [`docs/`](docs/README.md) | Architecture and operational knowledge |

## Boundaries

Six concerns, seven modules, one owner each. Every artifact has exactly one owning module. A change may coordinate artifacts across several modules when necessary; when it does, the reason belongs in the pull request.

```text
SOURCE CODE           backend/  frontend/   what the system does
BACKEND CONFIG        config/               how a backend application behaves
DEPLOYMENT STATE      gitops/               how and where it runs on Kubernetes
CLOUD INFRASTRUCTURE  infra/                what underlying infrastructure exists
AUTOMATION            .github/              how changes are validated and delivered
KNOWLEDGE             docs/                 why the system is shaped this way
```

The distinction that matters most:

```text
config/  → how a backend application behaves
gitops/  → how and where an application runs in Kubernetes
infra/   → what underlying infrastructure exists
```

## Environments

```text
local   local development on a developer's machine
dev     remote Kubernetes cluster
stage   remote Kubernetes cluster
prod    remote Kubernetes cluster
```

`dev`, `stage`, and `prod` are remote Kubernetes clusters: [`infra/`](infra/README.md) provisions them and [`gitops/`](gitops/README.md) declares what runs on them. `local` is not a cluster declared in this repository; how it is run on a developer's machine is not yet decided.

Environments are isolated. A change intended for one must not implicitly alter another.

## External systems

These are runtime systems BidPoint depends on. They may be **provisioned or configured** from this repository, but their runtime state and data never live in Git.

| System | Role |
| --- | --- |
| ECR | Built container artifacts |
| S3 / CDN | Built frontend artifacts — if static hosting is adopted; the web deployment model is undecided |
| AWS Secrets Manager / Vault | Secrets |
| EKS | Workload runtime |
| RDS / databases | Persistent data |
| Kafka / RabbitMQ | Messaging infrastructure |
| Observability platforms | Runtime telemetry |

## Working in this repository

Read [`AGENTS.md`](AGENTS.md) before making changes. Read the module's own README before changing a module. Record significant architectural decisions under `docs/adr/`.
