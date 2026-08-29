# backend/

The backend engineering domain: server-side application code implementing BidPoint's business capabilities.

## What belongs here

- Independently deployable services
- Domain and business logic
- APIs exposed to clients and to other services
- Persistence logic and database migrations
- Asynchronous producers and consumers
- Integrations with external providers
- Shared backend libraries

## What does not belong here

- Deployment infrastructure — that is [`gitops/`](../gitops/README.md) and [`infra/`](../infra/README.md)
- Environment-specific configuration, including hostnames, endpoints, and credentials
- Plaintext secrets of any kind

Backend code consumes configuration through explicit configuration interfaces and environment variables. Environment knowledge is supplied to a service at runtime; it is never embedded in the code. A service should not be able to tell which environment it is running in by reading its own source.

## Interacts with

- [`config/`](../config/README.md) — supplies the runtime settings a service reads
- [`gitops/`](../gitops/README.md) — declares how built images are deployed and scaled
- [`.github/`](../.github/AUTOMATION.md) — builds, tests, and publishes container artifacts to ECR
- [`frontend/`](../frontend/README.md) — consumes the APIs defined here

## Likely later

Service directories, shared libraries, API contracts, migration sets, and per-service build configuration. Service decomposition is deliberately not decided here — see [`docs/architecture`](../docs/README.md) and `docs/adr/` when it is.
