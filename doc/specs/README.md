# Scope Sage — Specification Index

This folder contains the complete architectural specification for Scope Sage, a GitHub bot that performs architectural pre-work on issues that have passed triage.

## Purpose

Scope Sage reads internal documentation (architecture docs, ADR/RFC catalogue, roadmap) and synthesises a structured architecture assessment document, posted as a comment on the issue. It sits between Triage Titan and CogWorks in the issue pipeline, gated on both sides by human review.

## Pipeline Context

```
[Triage Titan]       automated triage
     ↓
[Human gate]         reviews triage output
  applies → state:design
     ↓
[Scope Sage]         reads internal docs, synthesises architecture assessment
  posts structured comment on issue
     ↓
[Human gate]         reviews assessment
  applies → state:implementation
     ↓
[CogWorks]           implements the work
```

## Spec Files

| File | Contents |
|---|---|
| [overview.md](overview.md) | System context, pipeline position, high-level data flow |
| [vocabulary.md](vocabulary.md) | Domain concepts and definitions |
| [responsibilities.md](responsibilities.md) | Component responsibilities (RDD) |
| [architecture.md](architecture.md) | Clean architecture: business logic, abstractions, infrastructure |
| [assertions.md](assertions.md) | Testable behavioural assertions |
| [assumptions.md](assumptions.md) | Challenged assumptions and resolutions |
| [constraints.md](constraints.md) | Implementation rules and hard constraints |
| [security.md](security.md) | Threat model and mitigations |
| [testing.md](testing.md) | Test strategy and coverage requirements |
| [edge-cases.md](edge-cases.md) | Non-standard flows and failure modes |
| [tradeoffs.md](tradeoffs.md) | Design alternatives and rationale |
| [operations.md](operations.md) | Deployment, configuration, monitoring |

## ADRs

Architecture Decision Records live in [../adr/](../adr/).

| ADR | Decision |
|---|---|
| [ADR-0001](../adr/ADR-0001-hexagonal-architecture.md) | Trait-based abstractions for external system integration (domain-named traits) |
| [ADR-0002](../adr/ADR-0002-long-running-service.md) | Deploy as long-running service, not GitHub Action |
| [ADR-0003](../adr/ADR-0003-webhook-trigger.md) | Webhook-based trigger over label polling |
| [ADR-0004](../adr/ADR-0004-ed25519-document-signing.md) | Ed25519 signatures for output verification |
| [ADR-0005](../adr/ADR-0005-anthropic-llm-provider.md) | Anthropic Claude as LLM provider |
| [ADR-0006](../adr/ADR-0006-read-only-document-access.md) | Read-only access to internal documentation |

## Handoff Notes

**Interface Designer**: translate [vocabulary.md](vocabulary.md) into Rust types, [responsibilities.md](responsibilities.md) into trait definitions, and [architecture.md](architecture.md) into module boundaries. Use business domain names — no architectural layer names (`ports`, `adapters`, `core`) in module paths.

**Planner**: derive implementation tasks from [responsibilities.md](responsibilities.md) and [assertions.md](assertions.md). Each assertion cluster maps to a test suite. Each component maps to an implementation task.

**Coder**: read [constraints.md](constraints.md) before touching any file. It contains hard rules that CI enforces.
