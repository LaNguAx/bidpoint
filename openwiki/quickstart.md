---
type: repository quickstart
title: BidPoint Repository Quickstart
description: Evidence-based starting point for coding agents working in BidPoint's early-scaffolding monorepo, with mandatory reading, owner selection, safety checks, task routes, and current validation limits.
tags: [quickstart, repository, ownership, agent-guidance, scaffolding, validation]
verified:
  - by: openwiki/0.4.3
    at: 2026-08-29T17:10:15.382Z
sources:
  - id: openwiki-source-d7476156bc2e7db82971c90b
    resource: repo://.github/AUTOMATION.md
  - id: openwiki-source-1307a98427393d045f958ba3
    resource: repo://.github/CODEOWNERS
  - id: openwiki-source-1e075575622e1a77a3dc46e6
    resource: repo://.github/pull_request_template.md
  - id: openwiki-source-6d4b4e707b8d60b6ccfa3425
    resource: repo://.github/workflows/openwiki-update.yml
  - id: openwiki-source-ea70eb6c045047448e446296
    resource: repo://.gitignore
  - id: openwiki-source-8037e2358a2c4f9b2c722a11
    resource: repo://AGENTS.md
  - id: openwiki-source-9025181f12900b1c2ae4adf5
    resource: repo://backend/README.md
  - id: openwiki-source-a2371d6362e5db4bc834ad03
    resource: repo://CLAUDE.md
  - id: openwiki-source-492a0ca28ef350e203697356
    resource: repo://config/README.md
  - id: openwiki-source-196170e31ff8ec60a116165b
    resource: repo://docs/README.md
  - id: openwiki-source-3300f2dd8adb4f9d3123f304
    resource: repo://frontend/mobile/README.md
  - id: openwiki-source-ae1dbc927029f2cde98099cb
    resource: repo://frontend/README.md
  - id: openwiki-source-6f2ecc221c1da36706d06d33
    resource: repo://frontend/web/README.md
  - id: openwiki-source-cb918ecaa6a15e9633a657c6
    resource: repo://gitops/README.md
  - id: openwiki-source-862443b88cee5adeb9e4ba55
    resource: repo://infra/README.md
  - id: openwiki-source-23775c3de52f3ab95a13cb8b
    resource: repo://README.md
generated: { by: "openwiki/0.4.3", at: "2026-08-29T17:10:15.382Z" }
---

# BidPoint Repository Quickstart

BidPoint is intended to be an organization-level monorepo for one distributed auction system: application source, backend runtime configuration, Kubernetes desired state, cloud infrastructure definitions, automation, and durable knowledge live in one repository under separate owners.

> **Start with the current reality:** this repository is early scaffolding. Its READMEs establish intended boundaries, but almost all product application code, configuration sets, infrastructure, Kubernetes workload state, build systems, delivery automation, and tests are still absent. Do not describe a documented future component as running software, and do not invent commands or runtime flows to fill the gaps.

## Mandatory reading before editing

Read in this order:

1. [`AGENTS.md`](../AGENTS.md). It is the authoritative agent instruction file; [`CLAUDE.md`](../CLAUDE.md) only points to it.
2. The root [`README.md`](../README.md) for the monorepo thesis and top-level boundaries.
3. The owning module guide for **every** module the task will change. For a frontend task, read both [`frontend/README.md`](../frontend/README.md) and the relevant client guide: [`frontend/web/README.md`](../frontend/web/README.md) or [`frontend/mobile/README.md`](../frontend/mobile/README.md). For automation, the owning guide is [`.github/AUTOMATION.md`](../.github/AUTOMATION.md).
4. The current implementation and focused tests for the affected behavior, when they exist. Source and tests are authoritative.
5. The task-specific wiki page from the routing table below. OpenWiki is optional just-in-time context, not a replacement for current source, tests, or module documentation.

Treat `old/` as nonexistent: never read it, search it, cite it, or edit it.

## Choose the owner before choosing a file

Every artifact has one semantic owner. Directory proximity, file format, and the fact that several values eventually become environment variables do not change that owner.

| If the task changes… | Owning module | Read first |
| --- | --- | --- |
| Server-side business rules, APIs, persistence, migrations, async behavior, provider integrations, or backend libraries | `backend/` | [`backend/README.md`](../backend/README.md) |
| Browser UI, client state, browser integrations, assets, or non-sensitive web settings | `frontend/web/` | [`frontend/README.md`](../frontend/README.md), then [`frontend/web/README.md`](../frontend/web/README.md) |
| Mobile UI, native integrations, build profiles, or mobile packaging and release configuration | `frontend/mobile/` | [`frontend/README.md`](../frontend/README.md), then [`frontend/mobile/README.md`](../frontend/mobile/README.md) |
| Backend application behavior such as endpoints, feature flags, timeouts, retries, log levels, or business thresholds | `config/` | [`config/README.md`](../config/README.md) |
| Kubernetes workload operation such as image versions, replicas, resources, probes, networking, identities, or autoscaling | `gitops/` | [`gitops/README.md`](../gitops/README.md) |
| AWS accounts and environment topology, networks, clusters, IAM, registries, databases, KMS, or secret-manager resources | `infra/` | [`infra/README.md`](../infra/README.md) |
| CI/CD, build and release workflows, validation automation, review routing, or repository policy | `.github/` | [`.github/AUTOMATION.md`](../.github/AUTOMATION.md) |
| Long-lived architecture rationale, ADRs, diagrams, runbooks, or debugging guidance | `docs/` | [`docs/README.md`](../docs/README.md) |

The shortest diagnostic for the most easily confused modules is:

```text
config/  → how a backend application behaves
gitops/  → how and where an application runs in Kubernetes
infra/   → what underlying infrastructure exists
```

A change may coordinate several modules, but the pull request must identify all affected owners and explain why they need to move together. Coordination does not make a setting, manifest, workflow, or document jointly owned.

## Route the task through the wiki

The complete planned hierarchy is compact by design. Use the question that matches the task, then return to current source and the owning module README before editing:

- [`quickstart.md`](quickstart.md) — this starting page for reading order, owner selection, safeguards, and current validation limits.
- [`architecture/application-domains.md`](architecture/application-domains.md) — **Application Domains and Client Boundaries**: backend rule and API authority, independent web and mobile workspaces, API/configuration ownership, release isolation, and deliberately open application decisions.
- [`architecture/ownership-and-system-boundaries.md`](architecture/ownership-and-system-boundaries.md) — **Ownership and System Boundaries**: source-of-truth selection across application source, backend behavior, Kubernetes desired state, cloud foundations, automation, and durable knowledge, including intended runtime handoffs and failure boundaries.
- [`concepts/configuration-secrets-and-environments.md`](concepts/configuration-secrets-and-environments.md) — **Configuration, Secrets, and Environment Isolation**: placement of backend settings, workload controls, cloud resources, frontend settings, secret resources, references, and values; duplicate-owner prevention; and isolation across `local`, `dev`, `stage`, and `prod`.
- [`workflows/repository-change-lifecycle.md`](workflows/repository-change-lifecycle.md) — **Repository Change and Delivery Lifecycle**: scoping, pull-request review, focused validation, publication, desired-state change, reconciliation, and the distinction between intended product delivery and the one concrete OpenWiki workflow.

If the answer is not settled in current source or these ownership documents, that is an architectural gap—not permission to choose silently.

## Non-negotiable change rules

### One artifact and one setting owner

- Read the owning module's guide before changing it; do not infer responsibility from a directory name.
- Keep deployment and infrastructure concerns out of application modules, and keep application logic out of `gitops/` and `infra/`.
- Mobile build, signing, and store-release configuration is the narrow exception because it is inseparable from the mobile toolchain. The workflow that executes it still belongs in `.github/`.
- A backend setting meaningful to BidPoint code belongs in `config/`; a generic Kubernetes operating control belongs in `gitops/`; a cloud resource belongs in `infra/`; each frontend owns its own non-sensitive settings.
- For a backend workload, `gitops/` may set only `SPRING_PROFILES_ACTIVE`, `SPRING_APPLICATION_NAME`, and `SPRING_CONFIG_IMPORT`, plus variables sourced from Kubernetes Secrets. Another backend behavior variable there creates a second owner and can silently override the value in `config/`.

### No secrets

Never commit passwords, tokens, API keys, signing material, or credential-bearing connection strings anywhere—including examples, docs, `config/`, manifests, infrastructure definitions, or workflows. Secret values live in AWS Secrets Manager or Vault and are populated out of band. Git may contain an infrastructure declaration for a secret-manager resource or a workload reference to a secret, never the resolved value.

Frontend configuration is public by design: users can inspect browser output, mobile bundles, and device traffic. If a client integration appears to require a shared credential, change the system boundary instead of embedding the credential in the client.

The root `.gitignore` excludes common `.env` names, but that is only a convenience guard. It does not detect secrets under other names, make `.env.example` safe for real values, or remove a committed value from Git history.

### Preserve environment isolation

The exact model has four environments: `local` is developer-machine execution and is not a repository-declared Kubernetes cluster; `dev`, `stage`, and `prod` are isolated remote Kubernetes environments. State the intended target, edit only declarations intentionally in scope, and verify that a target-specific change does not implicitly alter another environment. Backend source must receive environment knowledge through explicit configuration and environment variables rather than hardcoding it.

### Do not invent architecture

Keep open choices open. Backend service decomposition, the web hosting model, concrete product toolchains, infrastructure tooling and topology, contract publication, and several delivery mechanisms have not been implemented or fully decided. If a task requires such a choice:

1. stop treating it as routine implementation;
2. state the decision being required and the alternatives;
3. record the accepted decision under `docs/adr/`; and
4. update a module README if the module's responsibility changes.

An undocumented architectural decision is indistinguishable from an accident. Existing ADRs should be superseded rather than deleted so their reasoning remains available.

### Keep changes narrow and explain crossings

Touch only what the requested result requires. Avoid adjacent refactors, broad formatting, or abstractions for a single use. In the pull request, summarize what changed and why, list every owning module, explain necessary cross-module coordination, and call out any environment scope or ADR. The repository's pull request template makes these review points explicit.

## Current artifacts versus intended system

| Area | Present evidence | Intent that must not be reported as implemented |
| --- | --- | --- |
| Repository architecture | Root, agent, and module boundary documents | Concrete backend services, API contracts, persistence, messaging topology, and runtime request flows |
| Frontends | Separate `web/` and `mobile/` boundary READMEs | Client source, toolchains, dependency graphs, shared contracts, and path-specific client pipelines |
| Backend configuration | Ownership and delivery design in `config/README.md` | Configuration sets, per-environment values, Spring Cloud Config service, and refresh behavior in a running workload |
| Kubernetes delivery | Desired-state and environment rules in `gitops/README.md` | `apps/` or `platform/` trees, workload manifests, deployed controllers, and clusters being reconciled |
| Infrastructure | Resource ownership categories in `infra/README.md` | Terraform/OpenTofu code, environment topology, state, plans, applies, and provisioned resources |
| Governance | A wildcard [`CODEOWNERS`](../.github/CODEOWNERS) rule and the [pull request template](../.github/pull_request_template.md) | Per-module review owners or proof that branch protection enforces code-owner approval |
| Automation | The concrete [`openwiki-update.yml`](../.github/workflows/openwiki-update.yml) documentation updater | Product build, test, release, infrastructure, or GitOps validation workflows |

The current `CODEOWNERS` file routes every path to one default owner; it does not yet encode module-specific teams. Directory structure and the review checklist therefore express boundaries that human review must currently enforce.

The OpenWiki updater is the repository's only concrete workflow. It can be started manually or by its daily schedule, checks out full history, sets up Node 22, installs pinned OpenWiki and Mermaid tooling, runs `openwiki code --update --print`, and opens a reviewable update pull request limited to `openwiki`, `AGENTS.md`, `CLAUDE.md`, and its own workflow file. The job has content and pull-request write permissions and resolves API credentials through GitHub secret references; none of this is evidence of a BidPoint product build, test, infrastructure apply, GitOps validation, or deployment.

## Validation evidence available today

There is no repository-wide product build or test command to run, and no product test suite or deployable stack to exercise. Validation must match what actually changed:

1. **Start from authoritative material.** Check current files and any focused tests that exist when implementation is added. Treat planned wiki prose as context, not runtime evidence.
2. **Validate the owning boundary.** For a documentation-only scaffold change, review the rendered Markdown, links, ownership wording, and diff. For governance or workflow changes, inspect the exact template, ownership, trigger, permission, pin, path, and secret-reference changes.
3. **Use the narrowest quiet check.** When a module gains a toolchain, use its documented local validation rather than inventing a repository-wide command. Preserve complete failure output.
4. **Do not substitute one lifecycle for another.** A source build cannot prove Kubernetes desired state, an infrastructure plan cannot prove an apply, a merged GitOps declaration cannot prove reconciliation, and OpenWiki cannot prove product behavior.
5. **Check non-target scope.** For a future environment-specific declaration, render or plan all relevant targets and prove unchanged targets remain unchanged. No current pipeline does this automatically.

The intended product delivery boundary is nevertheless explicit: CI will build and publish artifacts; cluster deployment will occur by changing `gitops/` desired state and allowing a GitOps controller to reconcile it. CI must not deploy cluster workloads directly. Static web delivery, if that hosting model is selected, and mobile store releases are the non-cluster exceptions. This is an architectural contract for future automation, not a current operational path.

## Agent pre-flight checklist

Before completing a change, confirm:

- [ ] I did not read, search, cite, or edit `old/`.
- [ ] I read `AGENTS.md`, the root README, and every affected module guide.
- [ ] I separated present repository evidence from documented intent.
- [ ] Every changed artifact and setting has exactly one owner.
- [ ] I explained every necessary cross-module change.
- [ ] I committed no secret value and placed each reference with the correct owner.
- [ ] I stated the environment scope and preserved isolation across `local`, `dev`, `stage`, and `prod`.
- [ ] I did not hide a new architecture decision in implementation; an ADR records it when required.
- [ ] I kept the diff scoped and used only validation that the current repository can actually support.
