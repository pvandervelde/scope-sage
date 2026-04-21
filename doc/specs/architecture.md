# Architecture

Clean architecture boundaries for Scope Sage. Business logic is isolated from infrastructure. All external system dependencies are accessed through abstractions.

---

## Layering Principle

```
┌────────────────────────────────────────────────────────┐
│  Business Logic                                        │
│  (AssessmentOrchestrator, ContextAssembler,            │
│   DocumentRenderer, DocumentSigner, EventRouter)       │
│                                                        │
│  Depends only on: domain types + trait abstractions    │
└───────────────────────┬────────────────────────────────┘
                        │ depends on (via trait)
┌───────────────────────▼────────────────────────────────┐
│  External System Abstractions (Traits)                 │
│  (IssueTracker, IssueEventSource, DocumentRepository,  │
│   AssessmentEngine, SigningKeyStore, AuditLog,         │
│   FailureNotifier)                                     │
└───────────────────────┬────────────────────────────────┘
                        │ implemented by
┌───────────────────────▼────────────────────────────────┐
│  External System Implementations                       │
│  (OctocrabIssueTracker, GithubWebhookSource,           │
│   QueueEventSource, GitDocumentRepository,             │
│   AnthropicAssessmentEngine, EnvSigningKeyStore,       │
│   OtelAuditLog, ConfigurableFailureNotifier)           │
└────────────────────────────────────────────────────────┘
```

The dependency rule: business logic depends on trait abstractions. Concrete implementations implement those traits. Business logic never imports a concrete implementation directly.

---

## Business Logic

### Domain Types

All types that carry business meaning. These cross boundaries only as values (no trait objects required for the types themselves).

| Type | Description |
|---|---|
| `IssueRef` | Repository owner, name, and issue number |
| `Issue` | Full issue content: title, body, labels, metadata |
| `DesignLabel` | Constant: the string `"state:design"` |
| `AlignmentVerdict` | Enum: `Aligned` / `Misaligned` / `Unclear` |
| `AssessmentSections` | Six named section bodies returned by LLM |
| `AssessmentDocument` | Rendered Markdown string ready for posting |
| `DocumentHash` | SHA-256 digest (32 bytes) of issue title + body |
| `DocumentSignature` | Ed25519 signature bytes (64 bytes) |
| `SignatureBlock` | Formatted HTML comment string |
| `InternalDocumentSet` | Collection of loaded document contents with source metadata |
| `LlmContext` | Structured prompt: system instructions + issue + documents |
| `LabelEvent` | Qualified event: label applied, label = design label, target = issue |
| `AuditRecord` | Structured event record for the audit sink |
| `FailureNotice` | Formatted comment body for posting on failure |

### Business Operations

Operations that encode business rules. All are implemented in the business logic layer.

**Event qualification (EventRouter):**

- Accept only `issues.labeled` events
- Accept only events where the label name is `DesignLabel`
- Accept only events where the target is an issue (not a PR)

**Idempotency check (AssessmentOrchestrator):**

- Before starting an assessment, check whether the issue already has a comment from the Scope Sage bot account
- If found, record a `DuplicateAssessment` audit record and stop

**Context assembly (ContextAssembler):**

- Combine issue + documents into an `LlmContext`
- If the total token estimate exceeds the configured context window, apply a priority-based truncation: prefer architectural docs and ADRs over roadmap content, prefer recent documents over older ones

**Document hash computation (DocumentSigner):**

- Compute SHA-256 over `UTF-8(title + "\n" + body)` — both fields from the issue at the time of assessment

**Signature production (DocumentSigner):**

- Compute the `key-id`: first 16 hex characters of SHA-256 over the public key bytes corresponding to the signing key
- Sign the string `"approved:<ISO8601_timestamp>\nkey-id:<key-id>\nhash:sha256:<hex>"` with Ed25519 private key
- The timestamp used in the signature is the moment the comment is posted, not when processing began

**Failure decision (AssessmentOrchestrator):**

- On LLM failure: post `FailureNotice` on issue, send alert, write audit record
- On comment post failure: retry up to configured maximum, then send alert and write audit record
- On document load partial failure: log gap, continue with available documents, note gap in assessment

---

## External System Abstractions

Trait abstractions that isolate business logic from external systems. Each trait is defined in the business logic layer and implemented in a separate module.

### `IssueTracker`

```
async fn fetch_issue(issue_ref: IssueRef) -> Result<Issue, GithubError>
async fn find_assessment_comment(issue_ref: IssueRef) -> Result<Option<CommentId>, GithubError>
async fn post_comment(issue_ref: IssueRef, body: String) -> Result<CommentId, GithubError>
```

### `IssueEventSource`

```
async fn next_event() -> Result<LabelEvent, EventSourceError>
```

Abstraction over the event transport. Implementations emit `LabelEvent`s (a label was applied to an issue). Authentication and deserialisation are handled internally: `GithubWebhookSource` validates HMAC-SHA256 before emitting; `QueueEventSource` relies on queue broker authentication. The `EventRouter` (business logic) filters emitted events to those matching `DesignLabel`.

### `DocumentRepository`

```
async fn load_documents(sources: Vec<DocumentSource>) -> Vec<LoadResult>
```

Returns a `LoadResult` per source — either document content or a `DocumentLoadError`. Never fails the entire call on a single source failure.

### `AssessmentEngine`

```
async fn synthesise(context: LlmContext) -> Result<AssessmentSections, AssessmentEngineError>
```

Implementations handle retries internally up to a configured limit. Returns `AssessmentEngineError` if all retries fail.

### `SigningKeyStore`

```
async fn signing_key() -> Result<SigningKey, SigningKeyStoreError>
```

Returns a `SigningKey` containing the Ed25519 private key and the `key-id` derived from the corresponding public key. Called at the start of each signing operation. Private key material must never be cloned or stored beyond the signing operation.

### `AuditLog`

```
async fn record(event: AuditRecord) -> Result<(), AuditError>
```

Must not be called with a fire-and-forget pattern. If the write fails, the error must be logged (even if it cannot be surfaced further).

### `FailureNotifier`

```
async fn notify_failure(issue_ref: IssueRef, summary: String) -> Result<(), FailureNotifierError>
```

---

## External System Implementations

One implementation per abstraction, except `IssueEventSource` which has two. Implementations contain no business logic — they translate between the trait's domain operations and the concrete external system.

| Implementation | Implements | External System |
|---|---|---|
| `OctocrabIssueTracker` | `IssueTracker` | GitHub REST API v3 (via `octocrab`) |
| `GithubWebhookSource` | `IssueEventSource` | GitHub webhook HTTP endpoint (via `github-bot-sdk`) |
| `QueueEventSource` | `IssueEventSource` | Message queue (via `queue-runtime`) |
| `GitDocumentRepository` | `DocumentRepository` | Git repositories |
| `AnthropicAssessmentEngine` | `AssessmentEngine` | Anthropic Messages API |
| `EnvSigningKeyStore` | `SigningKeyStore` | Environment variable / secret file |
| `OtelAuditLog` | `AuditLog` | OTEL exporter |
| `ConfigurableFailureNotifier` | `FailureNotifier` | HTTP webhook or SMTP |

---

## Cross-Cutting Concerns

### Configuration

All configuration is loaded at startup from environment variables (see [operations.md](operations.md)). The service fails fast if required configuration is missing or malformed. No runtime reloading.

### Error Handling

Business errors are `Result<T, E>` values. Panics are not used in library code. The `AssessmentOrchestrator` is the single recovery point: it translates all downstream errors into either a `FailureNotice` or a `DuplicateAssessment` audit record.

### Observability

Every external call is wrapped in a `tracing` span. Every `AuditRecord` is emitted as a structured OTEL span event. No plain log strings — all log statements include structured fields (see [operations.md](operations.md)).

### Concurrency

The service handles concurrent label events from the configured `IssueEventSource`. `EventRouter` spawns a new `tokio::task` for each qualifying event, allowing assessment cycles to run concurrently without blocking the event loop. Each assessment cycle runs independently. The idempotency check (querying for existing comments via GitHub API) is eventually consistent — a very narrow race window exists where two concurrent events for the same issue could both pass the check and both post a comment. This is acceptable given that `state:design` is human-applied and the scenario is operationally unlikely. See [edge-cases.md](edge-cases.md#concurrent-label-events).
