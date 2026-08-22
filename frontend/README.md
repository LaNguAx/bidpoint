# frontend/

All user-facing BidPoint clients.

```text
frontend/
├── web/     browser applications
└── mobile/  mobile applications
```

Each subdirectory is intended to be an independent workspace root with its own toolchain, dependency graph, and CI path — none of which exist yet. `frontend/` is an organizational boundary, not a shared build system — there is no root package manifest or lockfile at this level, and adding one would take resolution control away from both clients.

## What belongs here

- Client application code, UI, and client-side state
- Client-specific integration code against BidPoint APIs
- Assets owned by a client

## What does not belong here

- Business rules that must be authoritative — those live in [`backend/`](../backend/README.md); a client may not be the only place a rule is enforced
- Deployment manifests or infrastructure definitions
- Secrets; browser and mobile bundles are readable by their users, so nothing confidential can ship inside them

## Interacts with

- [`backend/`](../backend/README.md) — the APIs clients consume
- [`config/`](../config/README.md) — non-sensitive runtime settings such as endpoints and feature flags
- [`.github/`](../.github/AUTOMATION.md) — path-aware build and deployment automation per client

## Cross-client rule

`web/` and `mobile/` must not import from each other. Where the two genuinely need to share something, it is shared as a versioned package or a common API contract — not a relative path across the boundary. Keeping shared contracts independent of either implementation is what allows the two to evolve on separate release cadences.
