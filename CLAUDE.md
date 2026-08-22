# CLAUDE.md

Instructions for AI agents working in this repository. This is the authoritative agent document; [`AGENTS.md`](AGENTS.md) points here.

## What this repository is

BidPoint is an organization-level monorepo holding the application source, runtime configuration, deployment state, cloud infrastructure, and automation for one distributed system. See [`README.md`](README.md) for the full thesis.

## Modules

| Path | Owns |
| --- | --- |
| [`backend/`](backend/README.md) | Server-side application systems |
| [`frontend/`](frontend/README.md) | User-facing clients — `web/` and `mobile/` |
| [`config/`](config/README.md) | Application runtime configuration |
| [`gitops/`](gitops/README.md) | Desired state of Kubernetes workloads |
| [`infra/`](infra/README.md) | Cloud and platform infrastructure |
| [`.github/`](.github/AUTOMATION.md) | CI/CD and repository automation |
| [`docs/`](docs/README.md) | Architecture and operational knowledge |

```text
config/  → how an application behaves
gitops/  → how and where an application runs in Kubernetes
infra/   → what underlying infrastructure exists
```

## Rules

**Read the module README before changing a module.** Each one states what belongs there and what does not. Do not infer a module's responsibility from its name.

**Respect separation of concerns.** A change belongs to exactly one module. If a change needs to span several, that is worth questioning before writing it; when it is genuinely necessary, state the reason in the pull request.

- No infrastructure or deployment concerns inside application modules. One exception: client release packaging inseparable from a client's toolchain — mobile build, signing, and store-release configuration — lives with that client; the workflows that execute it live in `.github/`.
- No application logic inside `gitops/` or `infra/`.
- No environment knowledge hardcoded in application code — services read configuration through explicit interfaces and environment variables.

**Never let a setting have two owners.** [`config/`](config/README.md) owns what an application does — timeouts, retries, feature flags, log levels, business thresholds. [`gitops/`](gitops/README.md) owns how it runs — replicas, image version, resources, probes, networking. A workload in `gitops/` may set only bootstrap and identity environment variables (`SPRING_PROFILES_ACTIVE`, `SPRING_APPLICATION_NAME`, `SPRING_CONFIG_IMPORT`) plus secret-backed variables from Kubernetes Secrets; any other environment variable there declares a property that `config/` already owns. When a setting is ambiguous: one that is generic to all software belongs to `gitops/`, one that is meaningful only to BidPoint's domain belongs to `config/`.

**Never commit secrets.** No passwords, tokens, API keys, or credential-bearing connection strings anywhere in this repository, including `config/`. Secret values live in AWS Secrets Manager or Vault; only references to them are declared in Git.

**Preserve environment isolation.** `dev`, `staging`, and `prod` are separate. A change intended for one must not implicitly alter another.

**Prefer small, scoped changes.** Touch only what the task requires. Do not refactor adjacent code, reformat untouched lines, or add abstractions for single-use code.

**Do not invent architecture.** If a task requires an architectural decision that has not been made, do not make it silently. State it, and record it under `docs/adr/` when it is decided. An undocumented decision is indistinguishable from an accident to whoever reads it next.

**Do not overstate the state of the system.** The repository is early scaffolding. Documentation describes intent; almost nothing is implemented. Do not write documentation that implies otherwise.

## Where to look

- [`README.md`](README.md) — repository thesis and boundaries
- Module `README.md` files — per-module responsibilities
- `docs/architecture/` — system architecture
- `docs/adr/` — Architecture Decision Records

## old/

**Ignore `old/` completely. It does not exist.**
Never read it, never search it, never cite it, never edit it.
