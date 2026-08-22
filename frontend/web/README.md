# frontend/web/

The browser-based BidPoint applications.

## What belongs here

- The main web application
- Administrative or internal interfaces
- Shared UI packages consumed by the applications in this workspace
- Browser-specific integration code
- Micro-frontends, if the architecture later adopts them

## What does not belong here

- Mobile application code — see [`../mobile/`](../mobile/README.md)
- Authoritative business rules
- Deployment manifests, infrastructure, or secrets

## Interacts with

- [`backend/`](../../backend/README.md) — API consumer
- [`config/`](../../config/README.md) — endpoints, feature flags, and other non-sensitive settings
- [`.github/`](../../.github/AUTOMATION.md) — build and deployment automation scoped to this path

## Deployment model

Undecided. Static hosting behind a CDN and a server runtime are both open; the choice depends on framework requirements that have not been settled. Do not encode an assumption either way in application code. When it is decided, record it under `docs/adr/`.
