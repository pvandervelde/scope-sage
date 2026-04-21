# ADR-0001: Trait-Based Abstractions for External System Integration

Status: Accepted
Date: 2026-04-15
Owners: scope-sage

## Context

Scope Sage interacts with multiple external systems at runtime:

- **GitHub** — reading issues and posting comments (`IssueTracker`)
- **Event transport** — receiving label events via webhook or message queue (`IssueEventSource`)
- **LLM API** — synthesising assessment documents (`AssessmentEngine`)
- **Documentation repositories** — loading architecture docs, ADRs, and roadmap (`DocumentRepository`)
- **Signing key store** — providing the Ed25519 signing key (`SigningKeyStore`)
- **Audit sink** — recording operation records (`AuditLog`)
- **Alert target** — notifying the reviewer on failure (`FailureNotifier`)

Each of these may use different technologies in different environments (local development, staging, production). Without a deliberate boundary, business logic would import external SDK crates directly (`octocrab`, `reqwest`, `git2`, `ed25519-dalek`), making unit testing without network access impossible and making technology substitution expensive.

The `credential-provider` infrastructure crate established this same pattern for credential management across the platform. Scope Sage adopts it consistently.

## Decision

For every external system interaction, define a Rust trait named after the **domain capability**, not the underlying technology. Business logic depends only on these traits. Concrete implementations are defined in separate modules and wired at the application entry point (`main.rs`). No business logic module imports a concrete implementation directly.

Trait names describe what the capability does for the domain:

- `IssueTracker` — not `GithubClient`
- `AssessmentEngine` — not `LlmApi`
- `DocumentRepository` — not `GitReader`
- `SigningKeyStore` — not `VaultKeyClient`

Business logic accepts dependencies as `Arc<dyn Trait + Send + Sync>` or as generic type parameters where needed for testability.

## Consequences

**Enables:**

- Unit testing business logic without any network or filesystem access (test doubles implement the traits)
- Technology substitution without touching business logic (swap the implementation, rewire in `main.rs`)
- Clear compile-time visibility of which external systems each component depends on
- Consistent pattern across the codebase — every external dependency follows the same structure

**Forbids:**

- Business logic modules importing implementation-specific crates (`octocrab`, `reqwest`, `git2`, etc.) directly
- Concrete implementations in the same module as business logic
- Traits named after technologies rather than domain capabilities

**Trade-offs:**

- Wiring in `main.rs` grows as more implementations are added — this is intentional and explicit
- `Arc<dyn Trait>` adds a small amount of indirection — acceptable for the testing and flexibility benefits

## Alternatives Considered

- **Direct SDK usage in business logic**: fast to write initially; impossible to unit test without network access; creates tight coupling to vendor-specific APIs and error types. Rejected.
- **Dependency injection framework (e.g. `shaku`)**: adds significant complexity for a service with a small, stable set of external systems. Rejected.
- **Single `ServiceDependencies` struct with concrete types**: easier to construct, but still couples business logic to concrete implementations at compile time. Rejected.

## Implementation Notes

- Traits are defined in the business logic modules, not in a separate crate.
- Concrete implementations live in separate sibling modules, one per external system.
- `main.rs` is the only place that constructs concrete implementations and wires them.
- For unit tests, implement the trait with a test-only struct returning controlled data. No mocking framework required.
- Naming rule (enforced by code review): if a trait name contains a technology name (`Github`, `Anthropic`, `Git`, `Otel`), it is wrong and must be renamed.

## Examples

**Business logic — depends only on the trait:**

```rust
pub struct AssessmentOrchestrator {
    issues: Arc<dyn IssueTracker + Send + Sync>,
    documents: Arc<dyn DocumentRepository + Send + Sync>,
    engine: Arc<dyn AssessmentEngine + Send + Sync>,
    audit: Arc<dyn AuditLog + Send + Sync>,
    // ...
}
```

**Main entry point — wires concrete implementations:**

```rust
// main.rs only
let issues = Arc::new(OctocrabIssueTracker::new(github_token)?);
let documents = Arc::new(GitDocumentRepository::new(sources)?);
let engine = Arc::new(AnthropicAssessmentEngine::new(api_key, model)?);
let orchestrator = AssessmentOrchestrator { issues, documents, engine, ... };
```

**Unit test — uses a test double:**

```rust
#[tokio::test]
async fn test_orchestrator_duplicate_detection_skips_llm_call() {
    let issues = Arc::new(FakeIssueTracker::with_existing_assessment());
    let engine = Arc::new(FakeAssessmentEngine::that_panics_if_called());
    // assert: no LLM call made when comment already exists
}
```

## References

- [architecture.md](../specs/architecture.md) — external system abstractions and implementations
- [responsibilities.md](../specs/responsibilities.md) — component responsibilities and collaborators
- `credential-provider-core` crate — established this pattern for credential management across the platform
