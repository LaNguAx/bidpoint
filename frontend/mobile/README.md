# frontend/mobile/

The mobile application domain.

## What belongs here

- Mobile application code and UI
- Navigation and platform-specific behavior
- Native integrations and device capability access
- Mobile build, signing, and release configuration

## What does not belong here

- Web application code — see [`../web/`](../web/README.md)
- Authoritative business rules
- Deployment manifests, infrastructure, or secrets

## Interacts with

- [`backend/`](../../backend/README.md) — API consumer
- [`config/`](../../config/README.md) — non-sensitive runtime settings
- [`.github/`](../../.github/README.md) — build and release automation scoped to this path

## Coupling

Mobile and web are expected to diverge in UI and in release cadence, since store review gates mobile releases in a way it does not gate web. Share business contracts and API definitions where practical; do not couple mobile implementation to web implementation. A change to a web component should never require a mobile release.
