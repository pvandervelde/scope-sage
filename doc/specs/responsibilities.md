# Component Responsibilities

Responsibility-Driven Design (RDD) cards for every Scope Sage component. Each component has clear knowing and doing responsibilities and identified collaborators.

Components are grouped by layer: [orchestration](#orchestration), [business logic](#business-logic), [external system abstractions](#external-system-abstractions), and [external system implementations](#external-system-implementations).

---

## Orchestration

### AssessmentOrchestrator

**Responsibilities:**

- Knows: the complete assessment lifecycle and the order of operations
- Knows: when a cycle is a duplicate and should be skipped
- Does: coordinates the full flow from `IssueRef` to posted `AssessmentDocument`
- Does: decides on the failure path (post failure notice, alert reviewer) when any step fails
- Does: ensures an `AuditRecord` is written regardless of outcome

**Collaborators:**

- `IssueTracker` — reads issue content; checks for existing Scope Sage comment; posts assessment and failure notices
- `DocumentRepository` — loads `InternalDocumentSet`
- `ContextAssembler` — builds `LlmContext`
- `AssessmentEngine` — synthesises `AssessmentSections`
- `DocumentRenderer` — renders `AssessmentDocument` body
- `DocumentSigner` — produces `SignatureBlock`
- `AuditLog` — records `AuditRecord`
- `FailureNotifier` — notifies designated reviewer on failure

**Role:** Coordinator. Holds no business logic itself; delegates all knowing and doing to collaborators.

---

## Business Logic

### ContextAssembler

**Responsibilities:**

- Knows: the LLM prompt template for assessment tasks
- Knows: how to combine issue content and documents into a well-structured prompt
- Does: assembles `LlmContext` from an issue and an `InternalDocumentSet`
- Does: truncates or summarises document content that exceeds the configured context window
- Knows: that token budgeting uses a character-based heuristic of 4 characters per token as a conservative estimate (avoids a runtime tokenizer dependency; errs on the side of under-filling the context window)

**Collaborators:** None (pure transformation — no I/O)

**Role:** Transformer. Takes data in, returns a new data structure. No side effects.

---

### DocumentRenderer

**Responsibilities:**

- Knows: the `AssessmentDocument` Markdown template
- Knows: the required section order and heading conventions
- Does: renders `AssessmentSections` into a complete `AssessmentDocument` body string
- Does: produces an explicitly empty section body (e.g. "None identified.") for sections with no content

**Collaborators:** None (pure transformation — no I/O)

**Role:** Transformer. Deterministic and side-effect-free.

---

### DocumentSigner

**Responsibilities:**

- Knows: the signing algorithm (Ed25519) and the `SignatureBlock` format
- Knows: the `DocumentHash` computation (SHA-256 over `title + "\n" + body`)
- Knows: the `key-id` computation (first 16 hex characters of SHA-256 over the public key bytes)
- Does: computes `DocumentHash` from issue title and body
- Does: produces `DocumentSignature` using the `SigningKey` from `SigningKeyStore`
- Does: formats the complete `SignatureBlock` HTML comment string

**Collaborators:**

- `SigningKeyStore` — retrieves the Ed25519 signing key and its `key-id`

**Role:** Signer. All cryptographic operations happen here; no other component touches signing material.

---

### EventRouter

**Responsibilities:**

- Knows: the `DesignLabel` constant
- Does: consumes `LabelEvent`s from `IssueEventSource`
- Does: filters to events where `label == DesignLabel` and target is an issue
- Does: extracts the `IssueRef` from qualifying events and routes them to `AssessmentOrchestrator`
- Does: discards non-qualifying events (records reason in the audit log)

**Collaborators:**

- `IssueEventSource` — source of incoming label events
- `AuditLog` — records receipt of every event (qualifying or not)

**Role:** Filter. Stateless. Spawns a `tokio::task` for each qualifying event and invokes `AssessmentOrchestrator::process` within that task. The event loop itself never blocks on assessment completion.

---

## External System Abstractions

These are the trait abstractions that separate business logic from concrete external systems. All implementations fulfil exactly one of these traits.

### IssueTracker

**Responsibilities:**

- Knows: how to communicate with the GitHub Issues API
- Does: fetches issue title, body, labels, and metadata by `IssueRef`
- Does: searches issue comments for an existing Scope Sage assessment comment
- Does: posts a new comment on an issue

**Consumers:** `AssessmentOrchestrator`

---

### IssueEventSource

**Responsibilities:**

- Knows: how to receive label events from the configured transport (webhook or queue)
- Does: emits `LabelEvent`s when a label is applied to a GitHub issue
- Does: handles transport-specific authentication and deserialisation internally

**Consumers:** `EventRouter`

---

### DocumentRepository

**Responsibilities:**

- Knows: how to access content from a `DocumentSource`
- Does: retrieves document content from a configured source by path or glob
- Does: returns partial results with error metadata if a source is unavailable

**Consumers:** `AssessmentOrchestrator`

---

### AssessmentEngine

**Responsibilities:**

- Knows: how to call the configured LLM API
- Does: sends `LlmContext` to the LLM and receives a text response
- Does: parses the structured response into `AssessmentSections`
- Does: retries on transient errors (rate limits, timeouts) up to a configured limit

**Consumers:** `AssessmentOrchestrator`

---

### SigningKeyStore

**Responsibilities:**

- Knows: where the Ed25519 signing key is stored
- Does: retrieves the `SigningKey` (private key bytes + `key-id`) on demand
- Does: never logs or exposes the raw key bytes outside this abstraction

**Consumers:** `DocumentSigner`

---

### AuditLog

**Responsibilities:**

- Knows: the `AuditRecord` schema
- Does: writes an `AuditRecord` to the configured audit sink
- Does: never fails silently — if the write fails, it logs a structured error

**Consumers:** `AssessmentOrchestrator`, `EventRouter`

---

### FailureNotifier

**Responsibilities:**

- Knows: the configured alert target (reviewer identity or notification channel)
- Does: sends a failure alert with the `IssueRef` and error summary to the configured target

**Consumers:** `AssessmentOrchestrator`

---

## External System Implementations

Concrete implementations of the external system abstractions above.

### GithubWebhookSource (implements `IssueEventSource`)

- HTTP listener using `github-bot-sdk` bound to a configured port
- Validates `X-Hub-Signature-256` header using HMAC-SHA256 before emitting any event
- Deserialises GitHub webhook JSON payload into `LabelEvent`
- Returns HTTP 401 on HMAC failure; HTTP 400 on schema mismatch

### QueueEventSource (implements `IssueEventSource`)

- Queue consumer using `queue-runtime`
- Subscribes to the configured topic at startup
- Deserialises queue messages into `LabelEvent`
- Authentication is the queue broker's responsibility

### OctocrabIssueTracker (implements `IssueTracker`)

- GitHub REST API v3 via the `octocrab` crate
- Authenticated with a GitHub App token or PAT loaded from the secrets store
- Handles GitHub API rate limiting and transient errors

### GitDocumentRepository (implements `DocumentRepository`)

- Reads files from git repositories (local clone or remote over HTTPS)
- Respects the file path scope configured per `DocumentSource`
- Returns `DocumentLoadError` metadata for individual unavailable sources

### AnthropicAssessmentEngine (implements `AssessmentEngine`)

- Anthropic Messages API via HTTPS
- Sends the assembled `LlmContext` as a structured prompt
- Parses the response into `AssessmentSections`; returns `AssessmentEngineError` if parsing fails

### EnvSigningKeyStore (implements `SigningKeyStore`)

- Reads the Ed25519 private key from a configured environment variable or mounted secret file
- Computes `key-id` from the corresponding public key at load time
- Validates key format at startup; fails fast if the key is missing or malformed

### OtelAuditLog (implements `AuditLog`)

- Emits `AuditRecord` as structured OTEL span events
- Writes to the configured OTEL exporter endpoint

### ConfigurableFailureNotifier (implements `FailureNotifier`)

- Configurable: HTTP webhook (e.g. Slack incoming webhook) or SMTP
- Sends a structured failure message containing `IssueRef`, timestamp, and error summary
