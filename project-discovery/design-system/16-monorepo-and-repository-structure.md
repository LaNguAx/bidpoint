# Monorepo and repository structure

Status: Canonical
Last validated: 2026-08-01

BidPoint is a future polyglot Nx monorepo with Maven as the independent Java build authority. This is target design only: the existing structure template describes a future shape, not an initialized workspace.

## Authority and guardrails

| Concern | Owner | Contract |
| --- | --- | --- |
| Java build, dependencies, tests, reactor | Maven Wrapper and Maven reactor | Direct Maven commands remain valid for every Java project. |
| Cross-language graph, affected execution, cache, task UX | Nx 23.1.0 | Orchestrates work; does not replace Maven's Java lifecycle. |
| Node workspace tooling/future frontend packages | pnpm 11.4.0 | Manages the Nx/Node toolchain only; frontend libraries are **Open**. |
| Maven inference in Nx | `@nx/maven` 23.1.0 | **Experimental:** adopt only behind the mandatory guard below. |
| AWS infrastructure | Terraform | Owns AWS resources, never Argo CD-owned Kubernetes workloads. |
| Charts | Helm | Packages applications/platform add-ons, not environment composition. |
| Environment desired state | `deploy/` and Argo CD | Lives on `main` in `dev`, `stage`, `prod`, never environment branches. |

The experimental-plugin guard is non-negotiable: direct Maven commands must remain reproducible; each Nx target documents and proves its Maven equivalent; upgrades require a compatibility check; thin custom Nx wrappers are the fallback if inference breaks. Maven is the escape hatch and authority.

## Future root contract

```text
BidPoint/
|-- apps/backend/
|   |-- api-gateway/  bidding-service/  payment-service/  realtime-service/
|   |-- notification-service/  notification-worker/  search-service/ (Staged)
|   `-- core-platform/             # intentional nested multi-module Maven exception
|-- libs/java/                      # cross-cutting mechanics only
|-- platform/                       # reusable platform definitions
|-- deploy/                         # GitOps composition: dev/stage/prod
`-- tests/  tools/  docs/
```

Backend applications are flat peers under `apps/backend`; `core-platform` is nested because its Spring Modulith modules are one deployable/local transaction boundary. Java root namespace is `com.bidpoint`.

Applications own deployable Helm charts and external REST/event contracts. External event contracts are producer-owned. `deploy/` composes charts and image digests; `platform/` owns reusable add-ons/policy; Terraform owns AWS resources. None creates a second owner for another boundary.

## Shared-code and image boundaries

Permitted shared Java libraries are cross-cutting mechanics only: `bidpoint-security-spring-boot-starter`, `bidpoint-observability-spring-boot-starter`, `bidpoint-web-spring-boot-starter`, `bidpoint-messaging-spring-boot-starter`, `bidpoint-reliable-messaging-spring-boot-starter`, and `bidpoint-test-support`. They provide security, observability, web, messaging, reliable-messaging, and test mechanics. Business-domain models, repositories, tables, and decision services are forbidden shared code; use a versioned contract, REST command, or producer-owned event instead.

Jib Maven Plugin 3.5.2 is the routine backend image path; routine per-service Dockerfiles are **Excluded**. Future pnpm/Nx UX (including `pnpm nx local-up core|platform`) is a design contract, not a present command, and must preserve a direct Maven equivalent, clear failure evidence, and ownership.

**Open:** frontend libraries, payment provider, notification delivery provider, MSK mode, schema registry, public license. **Staged:** search-service/frontend packages and custom Nx wrappers if needed. **Excluded:** Java builds that only work through Nx, routine Dockerfiles, environment branches, shared business models, and cosmetic microservices.
