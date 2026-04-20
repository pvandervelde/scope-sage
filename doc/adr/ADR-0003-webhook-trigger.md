# ADR-0003: Event-Driven Trigger Over Label Polling

Status: Accepted
Date: 2026-04-15
Owners: scope-sage

## Context

Scope Sage must activate when a human applies the `state:design` label to a GitHub issue. Two mechanisms exist for detecting this event: reacting to a pushed event notification, or polling the GitHub API periodically to detect label state changes.

Additionally, at the transport level, event notifications can arrive via two channels: a GitHub webhook (HTTP push from GitHub's servers) or a message queue (event emitted to an internal queue by an upstream system).

## Decision

Use an event-driven approach rather than polling. Scope Sage receives `LabelEvent`s through an `IssueEventSource` abstraction (see [ADR-0001](ADR-0001-hexagonal-architecture.md)). Two concrete implementations are available; the deployment operator configures which to use:

- **`GithubWebhookSource`** (via `github-bot-sdk`): listens for `issues.labeled` GitHub webhook events on an HTTPS endpoint. Validates `X-Hub-Signature-256` (HMAC-SHA256) before emitting any event. Returns HTTP 401 on validation failure.
- **`QueueEventSource`** (via `queue-runtime`): consumes events from a configured message queue topic. Authentication is handled by the queue broker; no webhook secret is required.

Both implementations emit `LabelEvent`s. The `EventRouter` (business logic) filters these to those where the label matches `DesignLabel`. Business logic is identical regardless of which source is configured.

Label **polling** is rejected regardless of transport.

## Consequences

- **Enables**: near-immediate response to label application (seconds, not minutes); no API rate limit consumption for detection; no state tracking between polls; deployment flexibility (webhook or queue depending on infrastructure).
- **Forbids**: polling the GitHub Issues API for label changes.
- **Trade-offs accepted**: the webhook source requires an HTTPS endpoint reachable by GitHub's infrastructure and HMAC validation. The queue source requires a running queue broker. GitHub delivers webhooks at-least-once, so idempotency handling is mandatory.

## Alternatives Considered

- **Label polling**: periodic calls to the GitHub API to list issues with the `state:design` label. Introduces latency proportional to the polling interval; consumes API rate limit budget continuously; requires tracking which issues have already been processed to avoid re-processing; more complex state management. Rejected.

- **GraphQL subscriptions**: GitHub's GraphQL API has subscription support, but it is an experimental feature with limited library support in Rust and an unstable contract. Not suitable for a production service.

## Implementation Notes

**`GithubWebhookSource` (webhook transport):**

- Implemented using `github-bot-sdk`.
- Webhook secret loaded from `SCOPE_SAGE_WEBHOOK_SECRET` at startup.
- HMAC-SHA256 validation uses constant-time equality (`subtle::ConstantTimeEq`) to prevent timing attacks.
- Returns HTTP 401 for invalid signatures, HTTP 400 for valid-but-malformed payloads, HTTP 200 for all qualifying and non-qualifying valid events.
- GitHub retries on non-2xx responses: 401 is not retried, preventing repeated processing of rejected spoofed requests.

**`QueueEventSource` (queue transport):**

- Implemented using `queue-runtime`.
- Queue connection credentials loaded from `SCOPE_SAGE_QUEUE_*` environment variables.
- Subscribes to the configured topic at startup; events are deserialized into `LabelEvent`s.
- Authentication is the queue broker's responsibility — no additional HMAC layer is applied.
- At-least-once delivery from the queue also requires idempotency handling (same as webhook path).

## References

- [GitHub webhook documentation](https://docs.github.com/en/webhooks)
- [ADR-0001](ADR-0001-hexagonal-architecture.md) — trait-based abstraction pattern (`IssueEventSource`)
- [security.md](../specs/security.md#threat-01-event-spoofing) — THREAT-01 and mitigations
- [edge-cases.md](../specs/edge-cases.md#duplicate-webhook-delivery) — duplicate delivery handling
