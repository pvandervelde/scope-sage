# ADR-0004: Ed25519 Signatures for Assessment Document Verification

Status: Accepted
Date: 2026-04-15
Owners: scope-sage

## Context

Scope Sage posts an assessment comment as a GitHub issue comment. Downstream automation (GateKeeper, CogWorks) must be able to verify that a comment was genuinely produced by Scope Sage before acting on it. Without verification, any actor with write access to the issue could post a comment mimicking the assessment format and deceive downstream automation.

A verification mechanism must satisfy:

1. Verifiers (GateKeeper, CogWorks) can verify authenticity without needing write access.
2. No verifier can forge a valid-looking document — verification and signing use different keys.
3. The mechanism is compact enough to embed in a GitHub issue comment.
4. It is implementable without external services or runtime dependencies.

## Decision

Every `AssessmentDocument` includes a hidden HTML comment (the `SignatureBlock`) containing:

- An ISO8601 timestamp (`approved`): the moment the comment was posted.
- A signing key identifier (`key-id`): first 16 hex characters of SHA-256 over the public key bytes. Downstream systems use this to look up the correct public key, enabling clean key rotation without time-windowed key validity logic.
- A SHA-256 hash (`hash`): computed over `UTF-8(title + "\n" + body)` of the issue.
- An Ed25519 digital signature (`signature`): the Ed25519 signature over `"approved:<timestamp>\nkey-id:<key-id>\nhash:sha256:<hex>"`, base64-encoded, produced with Scope Sage's private key.

The public key (and its `key-id`) is published and distributed to GateKeeper and CogWorks for verification.

```
<!-- scope-sage
approved: 2026-04-15T10:23:45Z
key-id: a3f9e2b1c4d780fa
hash: sha256:e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855
signature: <base64-encoded Ed25519 signature>
-->
```

## Consequences

- **Enables**: downstream forgery detection; Scope Sage public key can be published without exposing signing capability; compact representation (64-byte signature = 88 base64 characters); clean key rotation via `key-id` lookup without time-windowed key validity.
- **Forbids**: using only a timestamp or a shared HMAC secret for verification (HMAC with a shared secret gives any verifier the ability to forge).
- **Trade-offs accepted**: key pair must be generated, stored securely, distributed to downstream systems with their `key-id`, and eventually rotated. A key rotation runbook is required.

## Alternatives Considered

- **No verification**: simplest, but downstream automation cannot distinguish genuine assessments from fabricated comments. Rejected — this is a security requirement, not an optimisation.

- **HMAC-SHA256 with a shared secret**: symmetric — any service that can verify can also forge. Distributing the secret to multiple services increases its exposure. Rejected.

- **RSA-2048 signature**: same security model as Ed25519, but larger keys (2048 bits vs 256 bits), larger signatures (256 bytes vs 64 bytes), and slower signing/verification. No meaningful advantage for this use case. Rejected.

- **Attestation via a third-party service**: e.g., a transparency log or external signing service. Adds a runtime external dependency for every assessment; introduces latency and availability risk. Rejected.

## Implementation Notes

- Use the `ed25519-dalek` crate for key generation, signing, and verification.
- Private key is stored as a file path referenced by `SCOPE_SAGE_SIGNING_KEY_PATH`. Loaded at startup; zeroed from memory after each signing operation using the `zeroize` crate.
- `key-id` computation: `hex::encode(&sha256(public_key_bytes)[..8])` — 8 bytes = 16 hex characters. Stable across restarts as long as the key pair does not change.
- The `key-id` and public key are stored at `SCOPE_SAGE_SIGNING_PUBLIC_KEY_PATH`. Downstream systems (GateKeeper, CogWorks) maintain a `key-id → public_key` map. On rotation, add the new entry before deploying; retire the old entry after a retention window.
- Hash input: `title.as_bytes()`, then `b"\n"`, then `body.as_bytes()` — concatenated and hashed via `sha2::Sha256`.
- Signature input: `format!("approved:{}\nkey-id:{}\nhash:sha256:{}", timestamp.to_rfc3339(), key_id, hex_hash)`.
- Base64 encoding uses standard alphabet (RFC 4648), no line wrapping (`base64::engine::general_purpose::STANDARD`).

## Key Rotation Procedure

See [security.md](../specs/security.md#key-management).

## References

- [ed25519-dalek crate](https://docs.rs/ed25519-dalek)
- [security.md](../specs/security.md#threat-05-forged-assessment-comment) — THREAT-05
- [vocabulary.md](../specs/vocabulary.md#signing-concepts) — `SignatureBlock`, `DocumentHash`, `DocumentSignature`
