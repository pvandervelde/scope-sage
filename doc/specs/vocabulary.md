# Domain Vocabulary

All domain concepts used across the specification. Interface Designer: use these exact names for types, field names, and function signatures. Coder: do not introduce synonyms — if the spec says `IssueRef`, the code says `IssueRef`.

---

## Core Concepts

### Issue

A GitHub issue that has been through triage and labelled `state:design` by a human reviewer. Scope Sage operates only on issues, not pull requests.

- Identified by: `IssueRef`
- Contains: title (non-empty string), body (Markdown string, may be empty), labels (list of label names), author (GitHub login), repository reference, timestamps

### IssueRef

A fully-qualified reference to a GitHub issue.

- Fields: repository owner (string), repository name (string), issue number (positive integer)
- Example: `pvandervelde/my-project#42`

### DesignLabel

The GitHub label name that triggers Scope Sage. Constant value: `"state:design"`.

Applied exclusively by humans. Scope Sage never reads its own trigger by writing it.

### AssessmentDocument

The structured Markdown comment posted by Scope Sage on an issue after a successful assessment cycle.

- Contains: six named sections (in fixed order) and a `SignatureBlock`
- Attributed to Scope Sage (comment author is the bot account)
- Timestamped in the `SignatureBlock`

### AlignmentVerdict

The explicit strategic alignment conclusion. Exactly one of:

- `Aligned` — the issue is consistent with current product direction
- `Misaligned` — the issue conflicts with current product direction
- `Unclear` — insufficient information to determine alignment

### StrategicAlignment

Section 1 of an `AssessmentDocument`. Contains the `AlignmentVerdict` and a rationale paragraph referencing the roadmap, active OKRs, and any explicit "not now" signals.

### AffectedAreas

Section 2 of an `AssessmentDocument`. A list of named systems, services, modules, or interfaces likely touched by implementing the proposed change.

### ArchitectureFit

Section 3 of an `AssessmentDocument`. Assessment of whether the proposed change is consistent with the target architecture. Notes any migration work, shims, or intermediate steps required.

### RelevantDecisions

Section 4 of an `AssessmentDocument`. Links to existing ADRs and RFCs that govern the affected area. Explicitly notes if none exist — this may indicate a documentation gap that the reviewer should flag.

### CrossCuttingConcerns

Section 5 of an `AssessmentDocument`. Any safety, interface, certification, or inter-service concerns the implementer must be aware of. May be empty (noted explicitly).

### OpenQuestions

Section 6 of an `AssessmentDocument`. Unresolved architectural questions the reviewing engineer should answer before transitioning the issue to `state:implementation`.

---

## Signing Concepts

### SignatureBlock

A hidden HTML comment appended to every `AssessmentDocument`. Not visible in the GitHub UI but trivially parseable by downstream automation.

```
<!-- scope-sage
approved: <ISO8601 timestamp>
key-id: <first 16 hex chars of SHA-256 over the Ed25519 public key bytes>
hash: sha256:<hex of SHA-256 over UTF-8(title + "\n" + body)>
signature: <Ed25519 signature over "approved:<timestamp>\nkey-id:<key-id>\nhash:sha256:<hex>", base64-encoded>
-->
```

The `key-id` allows downstream systems (GateKeeper, CogWorks) to look up the correct public key from their key registry when verifying the signature. This supports key rotation without time-windowed key validity logic.

### DocumentHash

SHA-256 digest over the UTF-8 encoding of `title + "\n" + body` for the issue being assessed.

- Input: concatenation of issue title, literal newline, issue body — all UTF-8 encoded
- Output: 32-byte digest represented as 64 lowercase hex characters

### DocumentSignature

Ed25519 signature over the UTF-8 string `"approved:<ISO8601_timestamp>\nkey-id:<key-id>\nhash:sha256:<hex>"`, where `<hex>` is the `DocumentHash` and `<key-id>` is the signing key identifier. Base64-encoded (standard alphabet, no line wrapping).

### SigningKey

The combined signing material returned by `SigningKeyStore`. Contains the Ed25519 private key and the `key-id` (first 16 hex chars of SHA-256 over the public key bytes). The private key bytes must be zeroed after use; the `key-id` is non-secret and may be retained.

### SigningKeyPair

The Ed25519 key pair used by Scope Sage to sign assessment documents.

- Private key: held exclusively by Scope Sage; loaded from a secrets store at startup; never logged or included in any output
- Public key: published together with its `key-id` to downstream automation (GateKeeper, CogWorks). Multiple key-id/public-key pairs can be in the downstream key registry simultaneously to support rotation.

---

## Document Loading Concepts

### DocumentSource

A configured reference to one internal documentation repository. Specifies location (URL or local path) and scope (which file paths or glob patterns within the repository to include).

### InternalDocumentSet

The collection of documents successfully loaded from all configured `DocumentSource` instances for a given assessment cycle. May be partial if one or more sources are unavailable.

### LlmContext

The fully assembled input to the LLM for a single assessment. Combines:

1. A structured system prompt defining the assessment task and output format
2. The issue title and body
3. The `InternalDocumentSet` contents
4. Instructions to produce the six-section `AssessmentDocument` structure

The `LlmContext` never contains secrets, credentials, or operational system data.

### AssessmentSections

The structured data returned by the LLM, parsed into the six named sections before rendering into the `AssessmentDocument` template.

---

## Event Concepts

### LabelEvent

An event signalling that a GitHub label has been applied to an issue. Emitted by `IssueEventSource` implementations (either `GithubWebhookSource` or `QueueEventSource`). The `EventRouter` processes only `LabelEvent`s where the label name matches `DesignLabel`.

### EventSourceError

An error from the `IssueEventSource` abstraction — for example, a network failure, deserialization error, or authentication failure on the transport layer. Does not include events that were rejected before being emitted (e.g. HMAC failures are handled internally by `GithubWebhookSource` and never surface as `EventSourceError`).

### AuditRecord

A structured record of a significant system event. Written to the audit sink after every LLM call, comment post, failure, or duplicate detection.

- Fields: event type, timestamp (ISO8601), `IssueRef`, outcome (success / failure / duplicate), latency (milliseconds), error details (if applicable)

---

## Failure Concepts

### FailureNotice

An issue comment posted when Scope Sage cannot complete an assessment cycle. Clearly attributes the failure to Scope Sage and instructs the reviewer to retry manually or proceed without the document.

### WebhookValidationError

An HTTP request to `GithubWebhookSource` failed HMAC-SHA256 signature validation. The request is rejected with HTTP 401 before any payload is processed. This is internal to `GithubWebhookSource` and never surfaces to business logic.

### DocumentLoadError

One or more `DocumentSource` instances were unavailable. Assessment proceeds with whatever documents were successfully loaded; the gap is recorded in the `AuditRecord` and noted in the `AssessmentDocument`.

### AssessmentEngineError

The LLM API call failed (timeout, rate limit, API error, malformed response). Scope Sage posts a `FailureNotice` and records the error in the `AuditRecord`. No assessment comment is posted.

### CommentPostError

The GitHub API call to post the assessment comment failed. Scope Sage retries up to a configured maximum; if all retries fail, it records a `FailureNotice` via the `FailureNotifier`.

### DuplicateAssessment

The issue already has a Scope Sage assessment comment. No new comment is posted. The event is recorded as a no-op `AuditRecord`.

---

## GitHub Integration Types

### CommentId

A GitHub-assigned identifier for a posted issue comment. Used by the idempotency check to reference an existing Scope Sage assessment comment.

- Obtained from: GitHub API response when a comment is created or found in a comment search
- Semantics: opaque; only meaningful for passing back to GitHub API calls (e.g. to retrieve or delete the comment)

### GithubError

An error returned by `IssueTracker` when a GitHub API operation fails. Variants include:

- `NotFound` — the issue, comment, or resource does not exist (HTTP 404)
- `RateLimit` — GitHub API rate limit or secondary rate limit exceeded (HTTP 429 or HTTP 403 with retry-after header)
- `Network` — connection failure or timeout before a response was received
- `Auth` — authentication or authorisation failure (HTTP 401 or HTTP 403 without retry-after)
- `Unexpected` — any other GitHub API error including 5xx server errors; wraps the HTTP status and response body

---

## Document Loading Types

### LoadResult

The outcome of a single `DocumentSource` load attempt. One of:

- **Success**: contains the `DocumentSource` label, the document content as a UTF-8 string, and source metadata (URL, branch, matched path)
- **Failure**: contains the `DocumentSource` label and a `DocumentLoadError` describing the reason the source was unavailable

`DocumentRepository::load_documents` returns one `LoadResult` per `DocumentSource` provided, in the same order.

---

## Error Types

### SigningKeyStoreError

An error returned by `SigningKeyStore` when the signing key cannot be retrieved. Variants:

- `KeyNotFound` — the configured environment variable or file path is absent
- `InvalidKeyFormat` — the key material exists but cannot be parsed as a valid Ed25519 private key
- `IoError` — file system or environment access failure (wraps the underlying OS error)

### AuditError

An error returned by `AuditLog` when an `AuditRecord` cannot be written to the configured audit sink. Wraps the underlying transport or serialisation failure.

`AuditError` is not propagated to callers — it is logged and discarded. A failing audit sink must not block or abort an assessment cycle.

### FailureNotifierError

An error returned by `FailureNotifier` when the failure alert cannot be delivered. Wraps the underlying HTTP, SMTP, or transport failure.

`FailureNotifierError` is logged but not propagated — a failing notifier must not mask or replace the original assessment failure in the audit record.
