# Design Tradeoffs

Alternatives considered during architectural analysis. Each entry records the options evaluated, the rationale for the chosen design, and the tradeoffs accepted.

---

## Deployment Model: Long-Running Service vs GitHub Action

**Chosen**: Long-running service on internal infrastructure.

**Alternatives considered:**

| Option | Pros | Cons |
|---|---|---|
| GitHub Action | Zero infrastructure management, built-in GitHub token | No persistent inbound webhook; cannot access internal non-public docs; ephemeral — private keys cannot be held safely; no persistent connections |
| Serverless (Lambda/Cloud Run) | Low operational overhead | Cold start latency; harder to hold signing key securely; outbound-only networking may not reach internal doc repos |
| **Long-running service** (chosen) | Persistent webhook receiver; access to internal networks; signing key in memory; connection reuse | Infrastructure management required; requires process supervision and health monitoring |

**Tradeoffs accepted**: Operational complexity in exchange for required access characteristics. See [ADR-0002](../adr/ADR-0002-long-running-service.md).

---

## Trigger Mechanism: Webhooks vs Label Polling

**Chosen**: GitHub webhook events (`issues.labeled`).

**Alternatives considered:**

| Option | Pros | Cons |
|---|---|---|
| Webhook (chosen) | Immediate response; no API rate limit consumption for polling; GitHub push model | Requires public HTTPS endpoint; requires HMAC validation; requires delivery retry handling |
| Label polling | No inbound endpoint required; simpler networking | Latency (≥ polling interval); rate limit consumption proportional to repo activity; state tracking complexity |
| GraphQL subscriptions | Real-time, efficient | Experimental GitHub API feature; limited library support |

**Tradeoffs accepted**: An HMAC-validated public HTTPS endpoint is required. Webhook delivery is at-least-once, requiring idempotency handling. See [ADR-0003](../adr/ADR-0003-webhook-trigger.md).

---

## Output Verification: Ed25519 Signature vs Simpler Alternatives

**Chosen**: Ed25519 digital signature in a hidden HTML comment block.

**Alternatives considered:**

| Option | Pros | Cons |
|---|---|---|
| No verification | Zero complexity | Downstream automation (GateKeeper, CogWorks) cannot distinguish a genuine assessment from a forged one |
| HMAC-SHA256 with shared secret | Simple; fast | Shared secret must be distributed to all consuming services; any service that can verify can also forge |
| **Ed25519 signature** (chosen) | Public-key scheme: verifiers need only the public key; cannot forge; compact (64-byte signature) | Key management required; rotation procedure needed |
| RSA signature | Widely understood | 2048-bit minimum key size; larger signature; slower |
| Attestation via a third-party service | No key management in the bot | External dependency; latency; cost |

**Tradeoffs accepted**: Ed25519 key pair must be generated, distributed, and eventually rotated. The public key must be published to GateKeeper and CogWorks. See [ADR-0004](../adr/ADR-0004-ed25519-document-signing.md).

---

## LLM Provider: Anthropic vs Alternatives

**Chosen**: Anthropic Claude (initial model: claude-opus-4 or equivalent current flagship).

**Alternatives considered:**

| Option | Pros | Cons |
|---|---|---|
| **Anthropic Claude** (chosen) | Strong long-context document synthesis; structured output support; clear usage policies | Dependency on third-party API; potential cost at scale |
| OpenAI GPT-4 series | Widely used; extensive tooling | Similar cost profile; less strong on very long context at time of evaluation |
| Self-hosted open-weight model | No data sharing with third party; no API cost | Significant infrastructure cost; worse quality for complex synthesis tasks; GPU dependency |
| Multiple providers with routing | Resilience against single provider outage | Prompt engineering must produce consistent results across providers; much higher complexity |

**Tradeoffs accepted**: Dependency on Anthropic's API availability and pricing. Data processing agreement with Anthropic is required. Model is configured, not hardcoded — a future ADR can change the model without architectural redesign. See [ADR-0005](../adr/ADR-0005-anthropic-llm-provider.md).

---

## Document Access: Read-Only Read at Assessment Time vs Caching

**Chosen**: Read documentation fresh from configured sources on each assessment cycle (no cache).

**Alternatives considered:**

| Option | Pros | Cons |
|---|---|---|
| **Fresh read per assessment** (chosen) | Always current documentation; no cache invalidation problem; simpler | Higher latency per assessment; more network I/O; source availability affects each cycle |
| Local cache with TTL | Lower latency; source availability doesn't block reads | Cache invalidation complexity; stale documentation risk; more state to manage |
| Pre-indexed vector store | Semantic search over docs; faster retrieval | Significant infrastructure addition; embedding pipeline required; synchronisation complexity |

**Tradeoffs accepted**: Assessment latency is higher when loading large doc sets. The `DocumentRepository` abstraction allows a caching implementation to be introduced in a later iteration without changing business logic.

---

## Error Communication: Failure Notice on Issue vs Silent Failure vs Alert Only

**Chosen**: Post a `FailureNotice` comment on the issue AND send an alert to the configured reviewer.

**Alternatives considered:**

| Option | Pros | Cons |
|---|---|---|
| Silent failure (log only) | No noise on the issue | Reviewer may not notice; issue sits in `state:design` indefinitely |
| Alert only (no issue comment) | Less noise on the issue | Issue has no visible indication of what happened; reviewer must correlate alert to issue manually |
| **Issue comment + alert** (chosen) | Issue is self-documenting; reviewer is proactively notified; no ambiguity | Slightly more noise on the issue; comment posted even on transient failures |

**Tradeoffs accepted**: A `FailureNotice` appears on the issue even for transient LLM failures that might succeed on retry. The reviewer sees the failure, but can manually trigger a retry. This is preferable to silent failure.

---

## Idempotency Check: GitHub API Query vs Local State

**Chosen**: Query the GitHub issue comments API to check for an existing Scope Sage comment.

**Alternatives considered:**

| Option | Pros | Cons |
|---|---|---|
| **GitHub API query** (chosen) | Survives service restarts; no local state to manage | One additional API call per cycle; eventually consistent (race window) |
| In-memory set of processed issue refs | Fast; no API call | Lost on restart; no persistence across deployments |
| Persistent local store (SQLite, Redis) | Survives restarts; fast | Requires infrastructure dependency; state synchronisation if multiple instances |

**Tradeoffs accepted**: Narrow race window where concurrent events could produce two comments. Accepted as an operational non-issue given human-gated label application. See [edge-cases.md](edge-cases.md#concurrent-label-events).

---

## LLM Output Format: Structured JSON vs Freeform Markdown with Parsing

**Chosen**: Instruct the LLM to produce Markdown with well-defined section headings; parse into `AssessmentSections` by section heading.

**Alternatives considered:**

| Option | Pros | Cons |
|---|---|---|
| JSON output from LLM | Machine-parseable; strict schema | LLMs hallucinate JSON structure inconsistently; less natural for long prose sections; requires JSON mode support |
| **Markdown with section headings** (chosen) | Natural for document synthesis; LLMs produce consistent headings; human-readable raw output | Section heading parsing is slightly fragile; requires heading normalisation |
| XML-tagged sections | Unambiguous delimiters | Verbose; LLMs sometimes close tags incorrectly |

**Tradeoffs accepted**: Section parsing is based on heading text matching. The prompt specifies exact heading strings; the parser must handle minor whitespace variation. A malformed response that cannot be parsed into all six sections is treated as `LlmError::MalformedResponse` — no partial assessment is posted.
