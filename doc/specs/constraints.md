# Implementation Constraints

Hard rules that apply to all Scope Sage code. These supplement the project-wide rules in [.tech-decisions.yml](../../.tech-decisions.yml). Constraints here take precedence over assumptions, preferences, and convenience.

---

## Language and Toolchain

- Language: Rust, edition 2024
- Async runtime: `tokio` (multi-threaded)
- Minimum Rust version: tracked in `rust-toolchain.toml`

---

## Code Rules (from .tech-decisions.yml)

These are reproduced here for visibility. Violations fail CI.

- `unwrap()` and `expect()` are forbidden in library code. Permitted only in `main()` entry point and `#[test]` functions.
- `no_global_mutable_state`: no `static mut`, no `Mutex<T>` at module scope.
- All business errors are `Result<T, E>` values. Panics are not a control flow mechanism.
- `println!` / `eprintln!` are forbidden in library crates. Use `tracing::info!` and family.
- All log statements must include structured fields. No plain strings: `info!("processed issue", issue_number = %ref.number, repo = %ref.repo)`.

---

## Security Constraints

**MUST NOT violate — any violation is a blocker.**

1. **No hardcoded secrets.** API keys, the Ed25519 private key, and the webhook HMAC secret must be loaded from environment variables or a mounted secret file. They must never appear in source code, log output, or any serialised form.

2. **HMAC validation is mandatory and unconditional for the webhook event source.** `GithubWebhookSource` must validate `X-Hub-Signature-256` before emitting any event. The secret comparison must use a constant-time equality function (`subtle::ConstantTimeEq` or equivalent). Early-exit comparison is forbidden.

3. **Issue content must never be interpolated into shell commands.** Issue titles and bodies are untrusted user input. They are passed to the LLM as data within a structured prompt, not as commands or templates. No `std::process::Command` may include issue content.

4. **LLM context must never contain secrets or credentials.** Only documentation content and issue metadata may appear in the `LlmContext`. The LLM is treated as an untrusted external service.

5. **Ed25519 private key material must not be cloned, logged, or stored beyond the signing operation.** `SigningKeyStore` returns a `SigningKey`; the key bytes must be zeroed after use (using `zeroize` or equivalent). The `key-id` (public-key fingerprint) is non-secret and may be retained.

6. **`DocumentRepository` must only access configured sources.** No dynamic source URLs from issue content. The set of allowed sources is fixed at configuration load time. This prevents SSRF via issue content.

---

## Architecture Constraints

1. **Business logic never imports external system implementations directly.** The dependency graph must be: business logic → traits → implementations. The binary entry point (`main.rs`) is the only place that wires concrete implementations to trait abstractions.

2. **All external I/O must occur through a trait abstraction.** There must be no direct calls to `octocrab`, `reqwest`, or filesystem operations from within business logic modules.

3. **The `AssessmentOrchestrator` is the sole failure recovery point.** No other component posts comments, sends alerts, or writes audit records on its behalf without being directed to do so by the orchestrator.

4. **Every assessment cycle produces exactly one `AuditRecord`.** Multiple intermediate events (LLM call, comment post) are nested within the same record. Do not emit multiple top-level records per cycle.

---

## Sizing Constraints (from .tech-decisions.yml)

- Maximum function length: 50 lines
- Maximum file length: 500 lines
- Maximum cyclomatic complexity: 10

---

## Testing Constraints

- Business logic (orchestrator, context assembler, document renderer, document signer, event router): 100% line coverage
- Integration tests: 90% coverage
- External system implementations: 80% coverage
- Mutation score minimum: 70% (enforced by `cargo-mutants` in CI)
- Test naming: `test_{function}_{scenario}_{expected}`
- All tests must be deterministic. No `sleep` or time-dependent assertions without a mock clock.

---

## Dependency Policy

- Prefer standard library. Every added crate requires justification in the PR description.
- Minimum set of expected crates: `tokio`, `github-bot-sdk`, `queue-runtime`, `octocrab`, `reqwest`, `serde`, `serde_json`, `thiserror`, `anyhow`, `tracing`, `tracing-subscriber`, `opentelemetry`, `hmac`, `sha2`, `ed25519-dalek`, `zeroize`, `base64`, `subtle`, `hex`, `cargo-fuzz` (dev-only)
- No crates with `unsafe` blocks unless the crate is widely audited (e.g. `ed25519-dalek`, `subtle`)
- Run `cargo audit` weekly in CI; fail on `high` severity findings

---

## Documentation Constraints

- All `pub` types and functions require `///` doc comments
- Doc comments must include: purpose, error conditions, and an example for non-trivial APIs
- ADRs are required for: new architecture decisions, LLM model changes, key rotation procedures, changes to the signature format

---

## Retry Policy

All components that perform retries (`AnthropicAssessmentEngine`, `OctocrabIssueTracker`) must follow this backoff policy unless the external API's `Retry-After` response header specifies a different interval:

| Parameter | Value |
|---|---|
| Initial delay | 1 second |
| Backoff multiplier | 2× (exponential) |
| Maximum delay per attempt | 30 seconds |
| Jitter | ±10% of computed delay (prevents thundering herd) |
| Maximum attempts | Configurable via `SCOPE_SAGE_LLM_MAX_RETRIES` / `SCOPE_SAGE_COMMENT_MAX_RETRIES`; default 3 |

When an API response includes a `Retry-After` header (GitHub rate limit, Anthropic rate limit), the header value overrides the computed backoff delay for that attempt.

Retry logic must be implemented inside the external system implementation — never in business logic modules.
