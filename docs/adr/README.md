# Architecture Decision Records (ADRs)

This directory contains Architecture Decision Records (ADRs) for the ODEI project.

## What is an ADR?

An Architecture Decision Record (ADR) captures an important architectural decision made along with its context and consequences. ADRs create a decision log that helps:

- New team members understand why the system is the way it is
- Current team members remember the reasoning behind past decisions
- Future architects evaluate whether decisions are still valid
- Document trade-offs and alternatives considered

## ADR Format

Each ADR follows the template in [000-template.md](./000-template.md) and includes:

- **Context**: The situation and problem being addressed
- **Decision**: What was decided and why
- **Consequences**: The impact (positive, negative, and neutral)
- **Alternatives**: Other options that were considered
- **Implementation Notes**: Guidance for applying the decision

## Status Definitions

- **Proposed**: Under discussion, not yet decided
- **Accepted**: Decision made and being implemented
- **Deprecated**: No longer recommended but still in use
- **Superseded**: Replaced by a newer ADR (link to replacement)

## Index of ADRs

| ADR                                     | Title                                     | Status   | Date       |
| --------------------------------------- | ----------------------------------------- | -------- | ---------- |
| [001](./001-singleton-pattern.md)       | Singleton Pattern for Core Services       | Accepted | 2025-12-13 |
| [002](./002-freeze-detection.md)        | Main Thread Freeze Detection Architecture | Accepted | 2025-12-13 |
| [003](./003-mcp-server-architecture.md) | MCP Server Design Principles              | Accepted | 2025-12-13 |
| [004](./004-state-management.md)        | Centralized State Management with Store   | Accepted | 2025-12-13 |

## Creating a New ADR

1. Copy `000-template.md` to `NNN-title.md` where NNN is the next number
2. Fill in all sections with specific details
3. Set status to "Proposed"
4. Open for team discussion
5. Update status to "Accepted" when decision is final
6. Add entry to this README index

## Superseding an ADR

When an ADR needs to be replaced:

1. Create the new ADR documenting the new decision
2. Update the old ADR's status to "Superseded by ADR-NNN"
3. Update both ADRs to cross-reference each other
4. Update the index in this README
