# Testing Strategy

Test strategy for Scope Sage. All requirements here supplement the coverage targets in [.tech-decisions.yml](../../.tech-decisions.yml) and [constraints.md](constraints.md).

---

## Test Pyramid

```
                       ┌─────────────────┐
                       │   End-to-end    │  1–2 smoke tests (sandbox GitHub + real LLM)
                       └────────┬────────┘
                    ┌───────────┴───────────┐
                    │     Integration       │  One test per adapter, per happy + error path
                    └───────────┬───────────┘
              ┌─────────────────┴──────────────────┐
              │            Unit / Property          │  All business logic components
              └─────────────────────────────────────┘
```

---

## Unit Tests

Target: 100% line coverage on all business logic components.

### ContextAssembler

- Test: full `InternalDocumentSet` → `LlmContext` assembles all documents
- Test: empty `InternalDocumentSet` → `LlmContext` contains only issue content
- Test: total content exceeding configured context window triggers truncation
- Test: truncation prefers ADR/architecture docs over roadmap content
- Test: truncation prefers recent documents over older ones
- Property: `LlmContext` never contains the literal string of any configured secret value (property-based, fuzzing issue content)

### DocumentRenderer

- Test: all six sections present in correct order
- Test: empty section body renders explicit placeholder ("None identified.")
- Test: non-empty `AlignmentVerdict` is one of the three accepted values
- Test: section headings match the expected template exactly
- Test: rendered output is valid UTF-8 Markdown

### DocumentSigner

- Test: `DocumentHash` equals `SHA-256(UTF-8("title\nbody"))` for known inputs
- Test: empty body produces valid hash (not a panic)
- Test: `DocumentSignature` verifies successfully against the test public key
- Test: `SignatureBlock` contains all four required fields (`approved`, `key-id`, `hash`, `signature`)
- Test: `approved` timestamp in `SignatureBlock` is ISO8601
- Test: `key-id` in `SignatureBlock` is 16 hex characters matching the fingerprint of the test public key
- Test: modifying title after hash computation produces a different hash
- Test: signature produced with key-pair-A fails verification with key-pair-B (key isolation)
- Property: `sign(context) → verify(public_key, context) == Ok` for all well-formed inputs

### EventRouter

- Test: `LabelEvent` with label `state:design` on an issue → routes to orchestrator
- Test: `LabelEvent` with label `state:triage` → discarded, audit record written
- Test: `LabelEvent` targeting a pull request → discarded, audit record written

### AssessmentOrchestrator (orchestration logic)

Use test doubles for all external system abstractions.

- Test: successful full cycle → assessment comment posted, audit record written
- Test: idempotency check finds existing comment → no LLM call, audit record with `duplicate`
- Test: LLM failure → no assessment comment, failure notice posted, alert sent, audit record with `failure`
- Test: all document sources fail → failure notice posted, alert sent
- Test: one document source fails, others succeed → assessment proceeds, gap noted
- Test: comment post fails on all retries → alert sent, audit record with `failure`, no partial post
- Test: audit record is written even when the audit sink itself fails (errors are logged separately)

---

## Integration Tests

Target: 90% coverage across adapter code.

Each adapter test uses a real external system where feasible, or a contract-compatible stub.

### GithubWebhookSource (HMAC validation)

- Test: valid signature → payload deserialised, `LabelEvent` emitted
- Test: invalid signature (wrong secret) → HTTP 401, no event emitted
- Test: missing signature header → HTTP 401, no event emitted
- Test: correct signature but malformed JSON → HTTP 400, no event emitted
- Test: constant-time comparison (no timing difference between wrong-key and right-key wrong-body — measure statistically)

### QueueEventSource

Uses a test queue broker (in-process or fixtures).

- Test: message with label `state:design` on an issue → `LabelEvent` emitted
- Test: message with non-design label → `LabelEvent` emitted with correct label (filtering is `EventRouter`'s responsibility)
- Test: malformed queue message → `EventSourceError` returned, no panic
- Test: queue broker connection failure → `EventSourceError` returned

### OctocrabIssueTracker

Uses a GitHub sandbox repository (test organisation) or recorded HTTP fixtures.

- Test: `fetch_issue` returns correct title, body, labels for a known issue
- Test: `find_assessment_comment` returns `None` when no Scope Sage comment exists
- Test: `find_assessment_comment` returns `Some(id)` when Scope Sage comment exists
- Test: `post_comment` creates a comment and returns a valid `CommentId`
- Test: GitHub API rate limit response triggers retry with back-off

### AnthropicAssessmentEngine

Uses recorded HTTP fixtures (VCR pattern) to avoid live LLM costs in CI.

- Test: valid `LlmContext` → `AssessmentSections` with all six sections populated
- Test: API returns rate limit error → retried up to configured maximum
- Test: all retries exhausted → `AssessmentEngineError` returned
- Test: response missing a required section → `AssessmentEngineError` (malformed response)

### GitDocumentRepository

Uses local git repositories created in the test fixture.

- Test: load document matching configured glob → content returned
- Test: repository path does not exist → `DocumentLoadError` for that source only
- Test: repository is empty → empty `InternalDocumentSet` entry (not an error)
- Test: file outside configured scope is not returned

### EnvSigningKeyStore

- Test: valid private key in environment variable → key loaded successfully, `key-id` computed
- Test: missing environment variable → `SigningKeyStoreError` at startup
- Test: malformed private key bytes → `SigningKeyStoreError` at startup
- Test: `key-id` is deterministic: same key pair always produces the same `key-id`

---

## Property-Based Tests

Use `proptest` or `quickcheck`.

- `DocumentHash` is deterministic: same `(title, body)` always produces same hash
- `DocumentSigner::sign` followed by `verify` always succeeds for any `(title, body)` and any valid key pair
- `EventRouter` never panics on any `LabelEvent` input
- `DocumentRenderer` never produces a comment body missing any of the six section headings
- `ContextAssembler` never includes secret values in `LlmContext` output (fuzz issue title/body)
- `SignatureBlock` parser: for any output of `DocumentSigner`, parsing the block always recovers `approved`, `key-id`, `hash`, and `signature`

---

## Fuzz Tests

Use `cargo-fuzz` with LLVM libFuzzer. Fuzz targets live in `fuzz/fuzz_targets/`.

| Target | Input | Goal |
|---|---|---|
| `fuzz_document_hash` | Arbitrary `(title, body)` byte pairs | No panics; output is always 64 hex chars |
| `fuzz_signature_block_parse` | Arbitrary byte sequences | No panics when parsing `<!-- scope-sage ... -->` blocks |
| `fuzz_webhook_payload` | Arbitrary HTTP request bytes | `GithubWebhookSource` never panics; always returns 400/401/200 |
| `fuzz_llm_response_parse` | Arbitrary LLM response strings | `AnthropicAssessmentEngine` response parser never panics |
| `fuzz_queue_message_parse` | Arbitrary queue message bytes | `QueueEventSource` deserialiser never panics |
| `fuzz_context_assembler` | Arbitrary issue content + document sets | `ContextAssembler` never panics, never exceeds token limit in output |

**Seed corpus**: each fuzz target ships with a seed corpus of known-interesting inputs (valid messages, boundary sizes, Unicode edge cases, truncated inputs).

**CI integration**: fuzz targets run for a fixed time budget (30 seconds each) on merge to `master`. Crashes are filed as security issues automatically.

**Regression**: any crashing input found by fuzzing is added to the seed corpus as a regression case and covered by a named unit test.

---

## End-to-End Tests

Run against a live GitHub sandbox repository and the real Anthropic API. Not run in every CI invocation — run on pre-release builds and scheduled nightly.

- Smoke test: apply `state:design` label to a prepared sandbox issue → assessment comment appears within 60 seconds with all six sections and a valid signature block
- Failure smoke test: configure an invalid Anthropic API key → failure notice comment appears on issue within 60 seconds

---

## Test Infrastructure Requirements

- All unit tests: `cargo nextest run` in CI on every commit
- All integration tests: run in CI on every commit using fixtures/stubs except GitHub API (uses sandbox org)
- Property tests: run in CI on every commit with a fixed seed for reproducibility, and with a random seed nightly
- Mutation testing: `cargo mutants` on business logic modules; minimum 70% mutation score
- End-to-end: run nightly and on pre-release branches

---

## Test Naming Convention

All test functions must follow: `test_{function_or_component}_{scenario}_{expected_outcome}`

Examples:

- `test_event_router_design_label_on_issue_returns_qualifying_event`
- `test_document_signer_empty_body_produces_valid_hash`
- `test_webhook_receiver_invalid_hmac_returns_401`
