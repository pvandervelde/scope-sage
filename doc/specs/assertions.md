# Behavioural Assertions

Testable assertions about system behaviour. Each assertion follows Given/When/Then format. These drive both the test suite structure and the interface contracts.

---

## Event Reception

### ASSERT-001: Qualifying label event triggers assessment

- **Given**: A qualifying `LabelEvent` where the label is `state:design` and the target is an issue (not a PR), received from any configured `IssueEventSource`
- **When**: Scope Sage processes the event
- **Then**: An assessment cycle begins for the identified issue

### ASSERT-002: Webhook with invalid HMAC signature is rejected before emission

- **Given**: Any HTTP request to `GithubWebhookSource` where the `X-Hub-Signature-256` header does not match the expected HMAC-SHA256 of the payload
- **When**: `GithubWebhookSource` receives the request
- **Then**: The request is rejected with HTTP 401
- **And**: No `LabelEvent` is emitted
- **And**: No audit record is written for the (potentially malicious) payload content

### ASSERT-003: Non-design label event is silently discarded (webhook source)

> Webhook-source-specific. For queue source see ASSERT-003b.

- **Given**: A valid `GithubWebhookSource` delivery of `issues.labeled` where the label is not `state:design`
- **When**: Scope Sage processes the event
- **Then**: HTTP 200 is returned to GitHub
- **And**: No assessment cycle begins
- **And**: An audit record is written recording receipt and discard reason

### ASSERT-003b: Non-design label event is silently discarded (queue source)

- **Given**: A `LabelEvent` received by `QueueEventSource` where the label is not `state:design`
- **When**: `EventRouter` processes the event
- **Then**: No `LabelEvent` is forwarded to `AssessmentOrchestrator`
- **And**: An audit record is written recording receipt and discard reason

### ASSERT-004: Pull request label event is silently discarded (webhook source)

> Webhook-source-specific. For queue source see ASSERT-004b.

- **Given**: A valid `GithubWebhookSource` delivery of `pull_request.labeled` where the label is `state:design`
- **When**: Scope Sage processes the event
- **Then**: HTTP 200 is returned to GitHub
- **And**: No assessment cycle begins
- **And**: An audit record is written recording receipt and discard reason

### ASSERT-004b: Pull request label event is silently discarded (queue source)

- **Given**: A `LabelEvent` received by `QueueEventSource` where the target is a pull request (not an issue)
- **When**: `EventRouter` processes the event
- **Then**: No `LabelEvent` is forwarded to `AssessmentOrchestrator`
- **And**: An audit record is written recording receipt and discard reason

### ASSERT-005: Malformed webhook payload returns HTTP 400

- **Given**: A webhook request that passes HMAC validation but contains a JSON payload that does not conform to the expected schema
- **When**: `GithubWebhookSource` receives the request
- **Then**: HTTP 400 is returned
- **And**: An audit record is written with the parse error

### ASSERT-005b: Queue message with non-design label is silently discarded

- **Given**: A queue message received by `QueueEventSource` where the label is not `state:design`
- **When**: `EventRouter` processes the `LabelEvent`
- **Then**: No assessment cycle begins
- **And**: An audit record is written recording receipt and discard reason

---

## Idempotency

### ASSERT-006: Duplicate label event on assessed issue is a no-op

- **Given**: A valid `issues.labeled` webhook for an issue that already has a Scope Sage assessment comment
- **When**: Scope Sage processes the event
- **Then**: No new assessment comment is posted
- **And**: An audit record is written with outcome `duplicate`
- **And**: No LLM call is made

---

## Assessment Content

### ASSERT-007: Assessment includes a non-empty strategic alignment verdict

- **Given**: A successfully completed assessment cycle
- **When**: The `AssessmentDocument` is examined
- **Then**: The Strategic Alignment section contains exactly one `AlignmentVerdict` (`aligned`, `misaligned`, or `unclear`)
- **And**: The verdict is accompanied by a non-empty rationale

### ASSERT-008: Assessment includes a non-empty affected areas list

- **Given**: A successfully completed assessment cycle
- **When**: The `AssessmentDocument` is examined
- **Then**: The Affected Areas section is present and contains at least one entry or an explicit statement that no areas were identified

### ASSERT-009: All six sections are always present

- **Given**: A successfully completed assessment cycle
- **When**: The `AssessmentDocument` is examined
- **Then**: All six sections are present in the correct order: Strategic Alignment, Affected Areas, Architecture Fit, Relevant ADRs and RFCs, Cross-Cutting Concerns, Open Questions
- **And**: Sections with no content contain an explicit placeholder (e.g. "None identified.") rather than being omitted

### ASSERT-010: Assessment is clearly attributed to Scope Sage

- **Given**: A posted `AssessmentDocument`
- **When**: The issue comment is examined
- **Then**: The comment is posted by the configured Scope Sage bot account

---

## Document Signing

### ASSERT-011: Every posted assessment contains a valid signature block

- **Given**: A successfully completed assessment cycle
- **When**: The posted comment HTML is inspected for the `<!-- scope-sage` block
- **Then**: The block contains: `approved` (ISO8601 timestamp), `key-id` (16 hex chars), `hash` (sha256: + 64 hex chars), `signature` (base64-encoded)

### ASSERT-012: Signature is verifiable using the key-id to select the public key

- **Given**: A posted `AssessmentDocument` with a `SignatureBlock`
- **When**: The `key-id` is used to look up the corresponding public key in the key registry, the `DocumentHash` is recomputed from the issue title and body, and the signature is verified with Ed25519
- **Then**: Verification succeeds

### ASSERT-012b: Signature from a rotated-away key is still verifiable while the old key is retained in the registry

- **Given**: A posted `AssessmentDocument` signed with an old key (old `key-id`) that has been rotated out but is still in the key registry
- **When**: Verification is attempted using the `key-id` to select the old public key
- **Then**: Verification succeeds

### ASSERT-013: Hash covers only the issue title and body at assessment time

- **Given**: An issue whose title is `"Improve caching"` and whose body is `"Cache should be faster."`
- **When**: The `DocumentHash` is computed
- **Then**: The hash equals `SHA-256(UTF-8("Improve caching\nCache should be faster."))`

### ASSERT-014: Modifying issue content after assessment invalidates the signature

- **Given**: A valid `SignatureBlock` for an issue
- **When**: The issue title or body is changed after the assessment was posted
- **Then**: Recomputing the `DocumentHash` from the new content produces a different hash
- **And**: Signature verification using the new hash fails

---

## Failure Handling

### ASSERT-015: LLM failure results in a failure notice, not a partial assessment

- **Given**: The LLM API returns an error for all retry attempts
- **When**: The assessment cycle processes the LLM step
- **Then**: No assessment comment is posted
- **And**: A `FailureNotice` comment is posted on the issue
- **And**: A failure alert is sent to the configured reviewer
- **And**: An audit record is written with outcome `failure` and the LLM error details
- **And**: The issue remains in `state:design`

### ASSERT-016: Document source unavailability degrades gracefully

- **Given**: One configured `DocumentSource` is unavailable but others are reachable
- **When**: Documents are loaded for an assessment
- **Then**: Assessment proceeds using the successfully loaded documents
- **And**: The assessment document explicitly notes which sources were unavailable
- **And**: An audit record records the partial load

### ASSERT-017: All document sources unavailable results in failure notice

- **Given**: All configured `DocumentSource` instances are unavailable
- **When**: Documents are loaded for an assessment
- **Then**: No assessment comment is posted
- **And**: A `FailureNotice` is posted on the issue
- **And**: A failure alert is sent to the configured reviewer

### ASSERT-018: Comment post failure triggers retry then alert

- **Given**: The GitHub API consistently returns errors when posting the assessment comment
- **When**: All configured retries are exhausted
- **Then**: A failure alert is sent to the configured reviewer
- **And**: An audit record is written with outcome `failure` and retry count

---

## Audit

### ASSERT-019: Every completed assessment cycle produces an audit record

- **Given**: Any assessment cycle — successful, failed, or duplicate
- **When**: The cycle completes
- **Then**: Exactly one `AuditRecord` is written to the audit sink
- **And**: The record includes: event type, `IssueRef`, outcome, timestamp, and latency

### ASSERT-020: LLM calls are individually recorded in the audit log

- **Given**: A successful assessment cycle
- **When**: The audit log is examined
- **Then**: There is a record of the LLM call including: model name, prompt token count, completion token count, latency, and outcome
