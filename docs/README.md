# docs/

The durable engineering knowledge base: why BidPoint is shaped the way it is, and how to operate it.

Documentation here outlives any single implementation. If knowledge is only useful while one branch is open, it belongs in the pull request, not here.

## Intended organization

```text
docs/
├── architecture/  system architecture documentation
├── adr/           Architecture Decision Records
├── diagrams/      architectural and system diagrams
└── runbooks/      operational procedures, incident and debugging guidance
```

These directories are created as content requires them.

## What belongs here

- System architecture and module boundaries
- Decision records capturing a choice, its context, and the alternatives rejected
- Diagrams of system structure and runtime flow
- Operational procedures and debugging guidance

## What does not belong here

- Module-specific documentation that belongs in that module's own README
- Generated API reference
- Transient planning notes and task lists

## Architecture Decision Records

When a significant architectural decision is made, record it under `docs/adr/`. A decision that is made in code but never written down becomes indistinguishable from an accident to whoever reads it next — including a future agent.

An ADR states the context, the decision, and the alternatives that were rejected and why. Superseded ADRs are marked superseded rather than deleted; the reasoning that was later overturned is often the most useful part of the record.
