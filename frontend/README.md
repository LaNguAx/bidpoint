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
- Client-specific non-sensitive configuration, including endpoints and feature flags
- Assets owned by a client

## What does not belong here

- Business rules that must be authoritative — those live in [`backend/`](../backend/README.md); a client may not be the only place a rule is enforced
- Deployment manifests or infrastructure definitions
- Secrets; browser and mobile bundles are readable by their users, so nothing confidential can ship inside them

## Interacts with

- [`backend/`](../backend/README.md) — the APIs clients consume
- [`.github/`](../.github/AUTOMATION.md) — path-aware build and deployment automation per client

## Configuration ownership

Each client owns its configuration inside its own workspace. `web/` and `mobile/` do not read from the root [`config/`](../config/README.md) module, which is backend-only. Configuration may not contain secrets because frontend artifacts are readable by their users.

## Cross-client rule

`web/` and `mobile/` must not import from each other. Where the two genuinely need to share something, it is shared as a versioned package or a common API contract — not a relative path across the boundary. Keeping shared contracts independent of either implementation is what allows the two to evolve on separate release cadences.
