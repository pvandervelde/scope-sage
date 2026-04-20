# Edge Cases and Failure Modes

Non-standard flows, boundary conditions, and failure scenarios that the implementation must handle correctly. Each entry states the condition, the required behaviour, and any relevant assertion cross-references.

---

## Webhook Edge Cases

### Duplicate webhook delivery

**Condition**: GitHub re-delivers the same webhook event (GitHub retries on non-2xx responses; network issues may cause duplicates before Scope Sage responds).

**Required behaviour**: The idempotency check in `AssessmentOrchestrator` detects that the issue already has a Scope Sage comment. No second assessment is produced. An audit record with outcome `duplicate` is written.

**See also**: [ASSERT-006](assertions.md#assert-006-duplicate-label-event-on-assessed-issue-is-a-no-op)

---

### Concurrent label events on the same issue {#concurrent-label-events}

**Condition**: Two `issues.labeled` webhook deliveries for the same issue arrive and are processed concurrently (e.g., label removed and re-applied rapidly).

**Required behaviour**: Both cycles perform the idempotency check. Since the check queries GitHub (eventually consistent), both may proceed past the check before either posts a comment. The worst outcome is two assessment comments on the issue. The reviewer ignores the duplicate. An audit record is written for both cycles.

**Design acknowledgement**: No distributed locking is implemented. This is an accepted residual risk given that `state:design` is human-applied and repeated re-application is operationally unusual. Documented in [assumptions.md](assumptions.md).

---

### Label applied then immediately removed

**Condition**: A human applies `state:design` and then removes it before Scope Sage finishes processing.

**Required behaviour**: Scope Sage does not check the current label state during processing — it acts on the event at delivery time. If the label was valid when delivered, the assessment proceeds. Removing the label after delivery has no effect on the in-progress cycle.

**Note**: This is intentional. The webhook event is a point-in-time record of intention. If the reviewer changes their mind, they can disregard the posted assessment.

---

### Webhook payload exceeds size limit

**Condition**: An issue with an unusually large body causes the webhook payload to exceed a size limit imposed by the HTTP server.

**Required behaviour**: `GithubWebhookSource` returns HTTP 413 (Payload Too Large). An audit record is written. No processing occurs.

---

## Issue Content Edge Cases

### Empty issue body

**Condition**: The issue has a title but an empty or whitespace-only body.

**Required behaviour**: Assessment proceeds. `LlmContext` includes the title and an explicit note that the body is empty. The LLM is expected to produce a lower-confidence assessment; the Open Questions section should reflect the lack of detail. `DocumentHash` is computed over `title + "\n"` (empty body).

---

### Issue body with only whitespace or boilerplate

**Condition**: The issue body contains only template placeholders or boilerplate (e.g., "## Description\n\n## Steps to Reproduce").

**Required behaviour**: Same as empty body. Assessment proceeds with limited context. The LLM prompt is not modified to detect boilerplate — this is left for the human reviewer to assess.

---

### Very large issue body (>50 000 characters)

**Condition**: The issue body is unusually long (e.g., a full requirements document pasted inline).

**Required behaviour**: The issue body is included in its entirety in the `LlmContext` unless it would push the total context over the configured window. If truncation is required, the body is truncated from the end with an explicit truncation notice appended. `DocumentHash` is computed over the full (untruncated) title and body.

---

### Non-ASCII characters in issue title or body

**Condition**: The issue contains non-ASCII characters (Unicode, emoji, right-to-left text).

**Required behaviour**: All string handling uses UTF-8. `DocumentHash` is computed over the raw UTF-8 bytes. No normalisation (NFC, NFD) is applied — hash is computed over the literal bytes as returned by the GitHub API.

---

## Document Loading Edge Cases

### All document sources unavailable

**Condition**: None of the configured `DocumentSource` instances can be reached.

**Required behaviour**: Assessment fails. A `FailureNotice` is posted on the issue. An alert is sent to the configured reviewer. An audit record with outcome `failure` is written. See [ASSERT-017](assertions.md#assert-017-all-document-sources-unavailable-results-in-failure-notice).

---

### Partial document source failure

**Condition**: One or more sources fail, but at least one succeeds.

**Required behaviour**: Assessment proceeds using available documents. The assessment comment explicitly lists which sources were unavailable. An audit record notes the partial failure. See [ASSERT-016](assertions.md#assert-016-document-source-unavailability-degrades-gracefully).

---

### Document content exceeding context window

**Condition**: The combined length of all loaded documents would push the `LlmContext` above the configured token limit.

**Required behaviour**: `ContextAssembler` applies priority-based truncation:

1. Architecture documentation (highest priority — never truncated unless unavoidable)
2. ADRs and RFCs (high priority)
3. Roadmap and OKR documents (lower priority)
4. Within each tier: prefer more recently modified documents

Truncation is noted explicitly in the `LlmContext` system prompt so the LLM is aware that context may be incomplete.

---

### Document source returns binary files

**Condition**: A `DocumentSource` path glob matches binary files (images, PDFs, compiled artifacts).

**Required behaviour**: Binary files are skipped. Only valid UTF-8 text files are included. A warning is written to the audit record listing skipped files.

---

## LLM Edge Cases

### LLM returns a response missing required sections

**Condition**: The LLM response does not conform to the expected six-section schema (e.g., sections are missing, headings differ).

**Required behaviour**: `AnthropicAssessmentEngine` returns `AssessmentEngineError::MalformedResponse`. The orchestrator treats this as a full LLM failure: posts `FailureNotice`, sends alert, writes audit record. No partial assessment is posted.

---

### LLM rate limit exceeded

**Condition**: The Anthropic API returns a rate limit error (HTTP 429).

**Required behaviour**: The adapter retries with exponential back-off up to the configured maximum retry count. If all retries fail, `AssessmentEngineError::RateLimit` is returned to the orchestrator.

---

### LLM response contains prompt-injected instructions

**Condition**: Issue content containing prompt injection instructions causes the LLM to include unexpected content in its response.

**Required behaviour**: The response is parsed against the fixed `AssessmentSections` schema. Content that does not map to the expected structure is discarded. The six sections are extracted; anything else is ignored. The human reviewer is the final gate.

---

## GitHub API Edge Cases

### GitHub API rate limit during issue fetch or comment post

**Condition**: The GitHub REST API returns HTTP 429 or HTTP 403 (secondary rate limit) during issue fetch or comment post.

**Required behaviour**: The `OctocrabIssueTracker` retries with the retry interval indicated by the `Retry-After` or `X-RateLimit-Reset` response header up to the configured maximum.

---

### Comment post succeeds but confirmation response is lost

**Condition**: The GitHub API accepts the comment but the HTTP response is lost in transit (network partition), so the adapter treats the call as failed and retries.

**Required behaviour**: Retry posts the comment again. The idempotency check (query for existing comment) only runs at the start of a cycle, not between retries within a cycle. The result may be two identical comments on the same issue. This is an accepted residual risk — operationally rare, and the reviewer can delete the duplicate.

---

### Issue deleted between webhook delivery and processing

**Condition**: An issue is deleted after the `issues.labeled` event is delivered but before Scope Sage fetches its content.

**Required behaviour**: `IssueTracker::fetch_issue` returns a `GithubError::NotFound`. The orchestrator writes an audit record with outcome `failure (issue_not_found)` and takes no further action. No failure notice is posted (the issue no longer exists).

---

## Signing Edge Cases

### Signing key unavailable at startup

**Condition**: The Ed25519 private key is not present in the configured location.

**Required behaviour**: The service fails to start. A clear startup error is logged: `"Ed25519 signing key not found at <location>"`. No webhook events are accepted until the service is restarted with a valid key.

---

### Signing key rotation mid-operation

**Condition**: The signing key is rotated in the secrets store while an assessment cycle is in progress.

**Required behaviour**: The key is loaded at the start of each signing operation (not cached for the lifetime of the process). If the new key is loaded during a cycle, the resulting `DocumentSignature` is valid against the new public key. Downstream systems must support verifying against multiple public keys during the rotation window.

---

## Operational Edge Cases

### Service restart during active assessment cycle

**Condition**: The service process is killed or crashes while processing an assessment.

**Required behaviour**: The in-progress cycle is abandoned. No partial comment is posted (comment posting is the last step in the cycle). On restart, the next `issues.labeled` event triggers a fresh cycle. If the label is still applied, the idempotency check determines whether a comment was successfully posted before the crash.

---

### Audit sink unavailable

**Condition**: The OTEL exporter or audit sink is unreachable.

**Required behaviour**: The assessment cycle completes normally. The failed audit write is logged as a structured error to stderr. Assessment and failure notice behaviour is not affected by audit sink availability.
