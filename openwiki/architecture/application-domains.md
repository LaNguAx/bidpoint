---
type: architecture concept
title: Application Domains and Client Boundaries
description: Planned authority and dependency boundaries between the backend and the independent web and mobile clients, including API contracts, configuration ownership, release isolation, and unresolved architecture decisions.
tags: [architecture, application-domains, backend, web, mobile, api-contracts]
verified:
  - by: openwiki/0.4.3
    at: 2026-08-29T16:26:54.497Z
sources:
  - id: openwiki-source-d7476156bc2e7db82971c90b
    resource: repo://.github/AUTOMATION.md
  - id: openwiki-source-1e075575622e1a77a3dc46e6
    resource: repo://.github/pull_request_template.md
  - id: openwiki-source-8037e2358a2c4f9b2c722a11
    resource: repo://AGENTS.md
  - id: openwiki-source-9025181f12900b1c2ae4adf5
    resource: repo://backend/README.md
  - id: openwiki-source-492a0ca28ef350e203697356
    resource: repo://config/README.md
  - id: openwiki-source-3300f2dd8adb4f9d3123f304
    resource: repo://frontend/mobile/README.md
  - id: openwiki-source-ae1dbc927029f2cde98099cb
    resource: repo://frontend/README.md
  - id: openwiki-source-6f2ecc221c1da36706d06d33
    resource: repo://frontend/web/README.md
  - id: openwiki-source-cb918ecaa6a15e9633a657c6
    resource: repo://gitops/README.md
  - id: openwiki-source-23775c3de52f3ab95a13cb8b
    resource: repo://README.md
generated: { by: "openwiki/0.4.3", at: "2026-08-29T16:26:54.497Z" }
---

# Application Domains and Client Boundaries

BidPoint is an organization-level monorepo, but it is not intended to be one application workspace. The repository assigns server-side behavior to [`backend/`](../../backend/README.md), browser applications to [`frontend/web/`](../../frontend/web/README.md), and mobile applications to [`frontend/mobile/`](../../frontend/mobile/README.md). The common [`frontend/`](../../frontend/README.md) directory is an organizational boundary, not a shared build root.

> **Current status:** this is an intended architecture, not a description of running applications. The repository is early scaffolding. The application directories contain boundary documentation, while application source, concrete toolchains, dependency graphs, per-client CI paths, backend services, and deployment definitions have not been implemented.

## Intended structure

<!-- openwiki: mermaid parse failed and this diagram was converted to a text fence so it does not break rendering. Fix the diagram source and restore the mermaid fence. Parser error: Heuristic: an unescaped angle bracket inside a label breaks rendering; rephrase the label. -->
```text
flowchart LR
  Backend["backend/<br/>business rules and APIs"]
  Config["config/<br/>backend behavior settings"]
  GitOps["gitops/<br/>workload desired state"]
  Automation[".github/<br/>path-aware automation"]
  Contract["Future independent versioned<br/>package or contract"]

  subgraph ClientDomains["Independent client domains"]
    Web["frontend/web/<br/>web workspace"]
    Mobile["frontend/mobile/<br/>mobile workspace"]
  end

  Web -->|"consumes APIs"| Backend
  Mobile -->|"consumes APIs"| Backend
  Contract -.->|"shared by version"| Web
  Contract -.->|"shared by version"| Mobile
  Config -->|"supplies runtime settings"| Backend
  GitOps -->|"declares deployment and scaling"| Backend
  Automation -->|"build and delivery path"| Web
  Automation -->|"build and release path"| Mobile
  Automation -->|"build and publish path"| Backend
```

*Figure 1. Intended ownership and dependency relationships; the boxes are planned domains, not implemented runtime components.*

The diagram deliberately does not show a web-to-mobile dependency or a concrete request sequence. Neither is part of the intended design, and no endpoint, protocol, gateway, service call chain, or asynchronous topology exists to document yet.

## Domain authority

### Backend: API and business-rule authority

The planned backend domain owns server-side application behavior: domain and business logic, APIs exposed to clients or other services, external-provider integrations, persistence concerns, and asynchronous behavior. Web and mobile are API consumers. A rule that must be authoritative must be enforced by the backend; a client-side check may improve the user experience, but it must never be the only enforcement point.

This assignment prevents a browser or modified mobile client from becoming the authority merely because it contains UI validation. It also gives both clients one server-side source of truth when their interfaces or release schedules diverge.

The backend README lists independently deployable services as a future category, but it does not decide service count or boundaries. **Backend services, shared backend libraries, API contracts, persistence, migrations, and asynchronous producers or consumers are all categories reserved for future implementation; they are not present systems.** In particular, this page does not imply a monolith, a set of microservices, a database design, a message broker topology, or a runtime request flow. Backend service decomposition must be decided explicitly and recorded as an architecture decision rather than inferred from the directory name.

### Web: browser application authority

The web workspace is intended to own browser application code, administrative or internal interfaces, browser-specific integrations, web UI packages used within that workspace, assets, and web-specific non-sensitive settings. Micro-frontends are only a possible future extension; their use has not been selected.

The web domain does not own mobile code, backend runtime configuration, authoritative business rules, infrastructure, deployment manifests, or secrets. Shared UI inside `frontend/web/` remains web-owned and does not become a mobile dependency.

### Mobile: device application authority

The mobile workspace is intended to own mobile application code and UI, navigation, platform-specific behavior, native integrations, device capability access, mobile assets and non-sensitive settings, and build, signing, and store-release configuration inseparable from its toolchain. Automation that executes builds and releases belongs under `.github/`, scoped to the mobile path.

The mobile domain does not own web implementation, authoritative business rules, backend runtime configuration, infrastructure, deployment manifests, or secrets.

## Workspace and dependency isolation

`frontend/web/` and `frontend/mobile/` are separate workspace roots. Each is intended to have its own toolchain, dependency graph, lockfile or equivalent resolution state, and CI path. None of those mechanisms exists yet. There is intentionally no package manifest or lockfile at `frontend/`: adding a shared root build would transfer dependency-resolution control away from both clients.

The hard dependency rules are:

1. Web must not import mobile implementation.
2. Mobile must not import web implementation.
3. A relative path crossing from one client workspace into the other is invalid, even if it is convenient in the monorepo.
4. Material genuinely needed by both clients may be shared only as an independent, versioned package or common API contract. It must not be owned by either client implementation.

The repository has not selected the location, publication mechanism, schema language, or compatibility policy for such packages or contracts. Those details remain implementation decisions. The architectural requirement is independence and versioning: each client chooses when to consume a compatible release rather than being forced to release because files changed in the other workspace.

## Entrypoints and control boundaries

There are no implemented application entrypoints. At the intended boundary, web and mobile enter the server-side domain through APIs defined by the backend. Concrete API endpoints and their transport, authentication, error model, and versioning are not yet specified, so no hypothetical request flow should be treated as implemented behavior.

The other planned control boundaries are configuration and delivery:

- A backend application will consume behavior settings through explicit configuration interfaces and environment variables. Environment knowledge must be injected at runtime rather than hardcoded into backend source.
- Backend settings are intended to be read at startup and on refresh from a configuration service, allowing a setting to change without rebuilding or redeploying its consumer. No configuration sets or configuration-server workload exist yet.
- Kubernetes reads desired workload state from `gitops/` when creating workloads; this is distinct from an application reading its own behavior settings.
- Each client build reads only the non-sensitive settings owned within that client's workspace.
- Path-aware automation is intended to build, test, and release each application domain independently. Concrete per-module workflows are not implemented.

## Configuration, secrets, and environment ownership

A setting has one owner. Its owner is determined by the process that consumes it, not by whether it happens to be represented as an environment variable.

| Concern | Owning domain | Representative values |
| --- | --- | --- |
| Backend application behavior | `config/` | service and messaging endpoints, feature flags, timeouts, retries, behavior switches, log settings, and their per-environment values |
| Web behavior | `frontend/web/` | non-sensitive browser endpoints and feature flags |
| Mobile behavior and packaging | `frontend/mobile/` | non-sensitive endpoints, feature flags, build profiles, signing and release configuration |
| Kubernetes workload operation | `gitops/` | image versions, replicas, resources, probes, networking, autoscaling, and deployment strategy |
| Secret values | External secret-management system | passwords, tokens, API keys, and credential-bearing connection strings; only references may be committed |

The root [`config/`](../../config/README.md) domain is backend-only. Web and mobile must not consume it. Client configuration remains local because each client has a distinct build and delivery lifecycle. It must also remain non-sensitive: browser artifacts, mobile bundles, and device traffic are inspectable by users.

For backend workloads, `gitops/` may inject only bootstrap and identity variables—`SPRING_PROFILES_ACTIVE`, `SPRING_APPLICATION_NAME`, and `SPRING_CONFIG_IMPORT`—plus values sourced from Kubernetes Secrets. Other backend behavior variables belong in `config/`. Duplicating a property in both domains is particularly dangerous because environment-variable precedence can silently override the configuration-service value, leaving the apparently correct value in `config/` unused.

`dev`, `staging`, and `prod` are isolated environments. A change scoped to one must not implicitly change another. This applies independently to application behavior configuration and workload desired state.

## API and release lifecycle

A change may coordinate backend, contract, and client artifacts in one pull request, but coordination does not erase ownership. The safe sequence is conceptual rather than an implemented pipeline:

1. Change authoritative rules and API behavior in the backend domain.
2. If clients consume an independently distributed contract, evolve and version that contract without placing it under either client implementation.
3. Update web and mobile in their respective workspace roots as compatibility requires.
4. Build, test, and release each affected domain through its own path-aware automation.

Web and mobile intentionally evolve on different release cadences. Mobile distribution is gated by store review; web delivery is not. A change to a web component must never require a mobile release. Conversely, backend and contract evolution must account for independently released or already-installed clients. The repository has not yet selected an API compatibility or client-support policy, so a change must not silently assume simultaneous rollout.

## Deployment and delivery boundaries

Backend artifacts are intended to be built and published by CI, while cluster deployment is controlled by declared state in `gitops/` and reconciliation by a GitOps controller. CI does not directly deploy cluster workloads.

The web deployment model is deliberately undecided. Static hosting behind a CDN and a server runtime are both open, pending framework requirements. Application code must not encode either assumption. If static hosting is selected, delivery may occur directly outside the cluster; if the web application becomes a cluster workload, it follows the GitOps path. The eventual choice belongs in an ADR.

Mobile release packaging stays in the mobile workspace because it is part of the client toolchain; repository automation runs that packaging and release process. This exception does not permit mobile deployment concerns to leak into web or backend code.

## Invariants and failure modes

| Invariant | What fails when it is violated |
| --- | --- |
| The backend is the authoritative rule-enforcement domain. | A modified or stale client can bypass a rule, and web and mobile can disagree about valid behavior. |
| Web and mobile never import one another. | Dependency resolution and releases become coupled; a web-only change can force a mobile rebuild or release. |
| Shared material is independent and versioned. | One client implementation becomes the de facto owner and consumers cannot upgrade on separate cadences. |
| Each setting has exactly one owner. | Competing sources can produce environment-specific behavior that differs from the reviewed configuration. |
| Client settings contain no secrets. | Credentials become recoverable from distributed artifacts or observable traffic. |
| Backend source contains no environment knowledge. | The same artifact can no longer be promoted safely across environments. |
| Web code assumes neither static nor server deployment. | An undecided hosting choice is made accidentally in code and becomes expensive to reverse. |
| Environment changes remain isolated. | A change intended for `dev`, `staging`, or `prod` can alter another environment's behavior or workload. |

## Validation as implementation arrives

No application or boundary-focused test suite exists yet. When these domains gain code, the smallest useful checks should correspond directly to the boundaries above:

- backend domain tests for authoritative rules, including cases that do not rely on client-side validation;
- API or contract compatibility tests against every client version the eventual support policy promises;
- dependency-boundary checks that reject imports crossing between `frontend/web/` and `frontend/mobile/`;
- standalone builds from each client workspace, proving that neither relies on an implicit `frontend/` root build;
- path-scoped CI checks proving that a client-only change does not trigger unrelated application or infrastructure pipelines;
- configuration validation that rejects duplicate backend settings in `config/` and `gitops/`, disallowed workload environment variables, plaintext secrets, and cross-environment leakage.

Tooling for these checks should be chosen with the eventual implementations. Choosing a backend framework decomposition, contract technology, client build system, or web hosting model merely to create a test now would turn an open decision into an undocumented one.

## Open architectural decisions

| Decision | Current constraint |
| --- | --- |
| Backend service decomposition | No service boundaries or count have been chosen. Future service directories, backend libraries, contracts, persistence, migrations, and async components must follow an explicit decision. |
| Web deployment model | Static delivery and a server runtime remain open. Do not assume either in application code; record the selection in `docs/adr/`. |
| Shared package and API-contract mechanism | Sharing must be independent and versioned, but its location, format, publication, and compatibility policy are not selected. |
| Web composition | Micro-frontends may be adopted later but are not current architecture. Web-internal UI sharing remains inside the web workspace. |
| Concrete build and CI toolchains | Web, mobile, and backend paths are intended to be independent, but their toolchains and per-module workflows do not exist yet. |

## Related pages

- [Ownership and System Boundaries](ownership-and-system-boundaries.md)
- [Configuration, Secrets, and Environments](../concepts/configuration-secrets-and-environments.md)
- [Repository Change Lifecycle](../workflows/repository-change-lifecycle.md)
