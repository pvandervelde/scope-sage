# Challenged Assumptions

Assumptions identified during architectural analysis, challenged, and resolved. Resolutions are binding design decisions.

---

## Assumption: "The service can run as a GitHub Action"

**Challenged because**: GitHub Actions are ephemeral and cannot hold persistent secrets in memory, cannot receive inbound webhooks directly, and cannot maintain stable connections to internal documentation repositories that are not publicly accessible. The access requirements (internal doc repos, signing key) make serverless execution impractical.

**Resolution**: Deploy as a long-running service on internal infrastructure. See [ADR-0002](../adr/ADR-0002-long-running-service.md).

**Impact**: Requires infrastructure provisioning, health monitoring, and process supervision. Not a trivial deployment unit.

---

## Assumption: "Label polling is an acceptable trigger mechanism"

**Challenged because**: Polling the GitHub API for label state introduces latency (minimum polling interval), GitHub API rate limit consumption proportional to repository activity, and complexity in tracking state between polls. Webhooks are the idiomatic GitHub trigger mechanism.

**Resolution**: Use GitHub webhook events (`issues.labeled`) as the trigger. See [ADR-0003](../adr/ADR-0003-webhook-trigger.md).

**Impact**: Requires an HTTPS endpoint reachable by GitHub's webhook delivery infrastructure. HMAC validation is mandatory.

---

## Assumption: "A timestamp in the comment header is sufficient for downstream verification"

**Challenged because**: A timestamp is not a proof of authenticity. Any actor with write access to the issue could post a comment that looks like a Scope Sage assessment. GateKeeper and CogWorks need to verify that the document was actually produced and signed by Scope Sage before operating on it.

**Resolution**: Include a cryptographic Ed25519 signature in a hidden block. GateKeeper verifies the signature against the published public key before accepting the document. See [ADR-0004](../adr/ADR-0004-ed25519-document-signing.md).

**Impact**: Requires key generation, key distribution to downstream systems, and key rotation procedures.

---

## Assumption: "Scope Sage should use the same LLM as Triage Titan and CogWorks"

**Challenged because**: Triage Titan and CogWorks may use different models for different tasks. Scope Sage's task is document synthesis — long-context reading and structured output — which may favour different model characteristics. Locking to a shared model introduces unnecessary coupling.

**Resolution**: Scope Sage selects its own model, configured independently. Anthropic Claude is the initial choice due to its document synthesis quality and long-context support. See [ADR-0005](../adr/ADR-0005-anthropic-llm-provider.md).

**Impact**: Scope Sage has its own Anthropic API key and model configuration, separate from other pipeline components.

---

## Assumption: "Scope Sage should have write access to labels for bookkeeping"

**Challenged because**: The spec explicitly states that Scope Sage must not apply labels or transition issue state. Label writes increase the blast radius of a compromised bot token and create coupling with GateKeeper's enforcement model. Any bookkeeping need can be met through issue comments and the audit log.

**Resolution**: Scope Sage has no label write access. All state communication is through comments and the audit log. See [ADR-0006](../adr/ADR-0006-read-only-document-access.md) and the access model in [overview.md](overview.md).

**Impact**: Scope Sage cannot self-describe its state through labels. Monitoring must rely on the audit log and the presence/absence of its comment.

---

## Assumption: "Internal documentation is stable enough to load fresh for every assessment"

**Challenged because**: Documentation repositories may be large, slow to clone, or temporarily unavailable. If Scope Sage clones repositories on every assessment, latency could be high.

**Resolution**: This is accepted as a runtime concern for the initial implementation. Document loading is behind a `DocumentRepository` abstraction, allowing a caching implementation to be introduced later without changing business logic. Partial failures degrade gracefully (see [assertions.md](assertions.md#ASSERT-016)).

**Impact**: Initial implementation may have higher latency for document-heavy assessments. Architecture does not prevent a cached adapter in a later iteration.

---

## Assumption: "Concurrent assessments for the same issue are impossible"

**Challenged because**: The `state:design` label could theoretically be removed and re-applied by a human while an assessment is in progress, or two webhook deliveries could arrive close together. The idempotency check (querying GitHub for an existing comment) is not transactional.

**Resolution**: The race window is accepted. Two concurrent webhook deliveries for the same issue after the `state:design` label is applied are operationally very unlikely given that the label is human-applied. The worst outcome is two assessment comments on the same issue — the reviewer simply ignores the duplicate. Documented in [edge-cases.md](edge-cases.md#concurrent-label-events).

**Impact**: No distributed locking mechanism is introduced. Accepted residual risk.
