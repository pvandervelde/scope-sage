# ADR-0005: Anthropic Claude as LLM Provider

Status: Accepted
Date: 2026-04-15
Owners: scope-sage

## Context

Scope Sage uses an LLM to synthesise a six-section architecture assessment from an issue and a set of internal documents. The task requires:

1. Long-context reading: internal documentation sets may be 50 000–150 000 tokens.
2. Structured output: the response must be parsed into six named sections reliably.
3. Document synthesis quality: the assessment must be coherent and actionable — not a summary but an architectural analysis.
4. Production reliability: the API must be stable enough for a production bot.

The LLM provider is configured (not hardcoded) so it can be changed via a future ADR if requirements change.

## Decision

Use the Anthropic API as the LLM provider. The initial model is configured via `SCOPE_SAGE_ANTHROPIC_MODEL` (no hardcoded model name in source). The Anthropic Messages API is called with a structured system prompt and a user message containing the `LlmContext`.

Scope Sage does not use any LLM framework (LangChain, LlamaIndex, etc.) — it calls the Anthropic API directly via HTTP.

## Consequences

- **Enables**: long-context document synthesis; reliable structured output with well-designed prompts; a clear API contract (no framework abstraction overhead).
- **Forbids**: hardcoding a model name in source code; tight coupling to the Anthropic SDK (direct HTTP calls or a minimal client are preferred to avoid SDK versioning friction).
- **Trade-offs accepted**: dependency on Anthropic API availability; data processing agreement with Anthropic is required before deployment; cost is proportional to token usage; if Anthropic changes pricing or discontinues a model, a config change plus retest is required.

## Alternatives Considered

- **OpenAI GPT-4 series**: comparable quality; slightly weaker on very long contexts at the time of evaluation; similar cost profile. Not technically inferior — could be substituted via a config change if needed.

- **Self-hosted open-weight model (Llama 3, Mistral, etc.)**: eliminates third-party API dependency and data sharing; significantly worse quality for complex architectural synthesis tasks at the token counts involved; requires GPU infrastructure that is not currently available internally. Rejected for the initial version — may be reconsidered if infrastructure becomes available.

- **Multiple providers with routing / fallback**: improves resilience against single-provider outage; significantly increases prompt engineering complexity (consistent output format across providers); increases operational complexity. Deferred — the `AssessmentEngine` abstraction allows this to be introduced without changing business logic.

- **LLM framework (LangChain/LlamaIndex in Rust, e.g. `llm-chain`)**: abstracts provider; adds dependency churn risk as these crates are young; hides the API call details that matter for auditability. Rejected — direct API calls are more transparent and auditable.

## Implementation Notes

- Call the Anthropic Messages API via `reqwest` (async HTTPS client).
- API key loaded from `SCOPE_SAGE_ANTHROPIC_API_KEY` environment variable.
- Model loaded from `SCOPE_SAGE_ANTHROPIC_MODEL` environment variable.
- Retry strategy: exponential back-off on HTTP 429 (rate limit) and HTTP 5xx (server error), up to `SCOPE_SAGE_LLM_MAX_RETRIES` attempts.
- The system prompt instructs the LLM to produce exactly six Markdown sections with specified headings. The response is parsed by heading text — do not rely on JSON mode.
- Token counting: estimate prompt tokens before sending; if the estimate exceeds `SCOPE_SAGE_CONTEXT_MAX_TOKENS`, apply `ContextAssembler` truncation before the API call.
- Every LLM call is recorded in the audit log including: model, input token count, output token count, latency, and outcome.

## References

- [Anthropic Messages API](https://docs.anthropic.com/en/api/messages)
- [AssessmentEngine interface](../specs/architecture.md#assessmentengine)
- [security.md](../specs/security.md#threat-06-llm-data-exfiltration) — THREAT-06
- [tradeoffs.md](../specs/tradeoffs.md#llm-output-format-structured-json-vs-freeform-markdown-with-parsing)
