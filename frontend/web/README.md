# frontend/web/

The browser-based BidPoint applications.

## What belongs here

- The main web application
- Administrative or internal interfaces
- Shared UI packages consumed by the applications in this workspace
- Browser-specific integration code
- Web-specific non-sensitive configuration, including endpoints and feature flags
- Micro-frontends, if the architecture later adopts them

## What does not belong here

- Mobile application code — see [`../mobile/`](../mobile/README.md)
- Authoritative business rules
- Backend runtime configuration — see [`config/`](../../config/README.md)
- Deployment manifests, infrastructure, or secrets

## Interacts with

- [`backend/`](../../backend/README.md) — API consumer
- [`.github/`](../../.github/AUTOMATION.md) — build and deployment automation scoped to this path

## Configuration

Web configuration lives in this workspace. Its delivery may be build-time or runtime depending on the deployment model, but it must remain non-sensitive because browser artifacts are readable by their users.

## Deployment model

Undecided. Static hosting behind a CDN and a server runtime are both open; the choice depends on framework requirements that have not been settled. Do not encode an assumption either way in application code. When it is decided, record it under `docs/adr/`.
