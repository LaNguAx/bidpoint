# Archive

Superseded design material. Kept for history only.

`project-discovery/` was the original design corpus — around thirty interlocking documents with a status vocabulary, an ADR register, versioned baselines, and precedence rules between them. The thinking in it is sound, and most of it survives in [`docs/`](../docs/) in plainer form.

It was archived because maintaining it had become the work. Every stack decision triggered a cascade of consistency edits across the corpus, and that consumed the time that should have gone into building.

**Do not maintain anything in here. Do not treat it as current.** Where it disagrees with [`docs/`](../docs/), `docs/` is right — the archive still describes seven deployables, Nx, Jenkins, Istio, EKS-as-primary, and other decisions that were later reversed.

Read it only when you want the reasoning behind a decision and `docs/` doesn't carry it. The most useful file for that is `project-discovery/design-system/2.0/04-decision-delta.md`, which records what changed and why.
