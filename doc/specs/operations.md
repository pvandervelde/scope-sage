# Operations

Deployment model, configuration, monitoring, and operational procedures for Scope Sage.

---

## Deployment Model

Scope Sage runs as a **long-running service** on internal infrastructure. It is not a GitHub Action or a serverless function. See [ADR-0002](../adr/ADR-0002-long-running-service.md).

- Process: single Rust binary (`scope-sage`)
- Supervisor: systemd, Kubernetes Deployment, or equivalent
- Replicas: one replica (idempotency check is eventually consistent; multiple replicas accepted with rare duplicate-comment risk)
- Restart policy: always restart on non-zero exit
- Graceful shutdown: SIGTERM triggers in-flight cycle completion then exit (maximum 30-second drain window)

---

## Configuration

All configuration is loaded from environment variables at startup. The service fails fast on startup if any required variable is missing or unparseable. There is no runtime configuration reloading.

### Required Variables

| Variable | Description |
|---|---|
| `SCOPE_SAGE_EVENT_SOURCE` | Event source type: `webhook` or `queue` |
| `SCOPE_SAGE_GITHUB_APP_ID` | GitHub App ID for authentication |
| `SCOPE_SAGE_GITHUB_PRIVATE_KEY_PATH` | Path to GitHub App RSA private key file |
| `SCOPE_SAGE_SIGNING_KEY_PATH` | Path to Ed25519 private key file (PEM or raw bytes) |
| `SCOPE_SAGE_SIGNING_PUBLIC_KEY_PATH` | Path to Ed25519 public key file (published to downstream with `key-id`) |
| `SCOPE_SAGE_ANTHROPIC_API_KEY` | Anthropic API key |
| `SCOPE_SAGE_ANTHROPIC_MODEL` | Anthropic model name (e.g. `claude-opus-4-5`) |
| `SCOPE_SAGE_DOCUMENT_SOURCES` | JSON array of `DocumentSource` objects (see below) |
| `SCOPE_SAGE_ALERT_TARGET` | Alert target (HTTP webhook URL or SMTP address) |
| `SCOPE_SAGE_OTEL_EXPORTER_ENDPOINT` | OTEL gRPC or HTTP exporter endpoint |

**Required when `SCOPE_SAGE_EVENT_SOURCE=webhook`:**

| Variable | Description |
|---|---|
| `SCOPE_SAGE_WEBHOOK_SECRET` | HMAC-SHA256 secret for webhook signature validation |
| `SCOPE_SAGE_LISTEN_PORT` | HTTP port for the webhook receiver (default: `8080`) |
| `SCOPE_SAGE_LISTEN_ADDR` | Bind address for the webhook receiver (default: `0.0.0.0`) |

**Required when `SCOPE_SAGE_EVENT_SOURCE=queue`:**

| Variable | Description |
|---|---|
| `SCOPE_SAGE_QUEUE_URL` | Queue broker connection URL |
| `SCOPE_SAGE_QUEUE_TOPIC` | Topic or queue name to subscribe to |
| `SCOPE_SAGE_QUEUE_CREDENTIALS_PATH` | Path to queue broker credentials file |

### Optional Variables

| Variable | Default | Description |
|---|---|---|
| `SCOPE_SAGE_LLM_MAX_RETRIES` | `3` | Maximum LLM API retry attempts |
| `SCOPE_SAGE_COMMENT_MAX_RETRIES` | `3` | Maximum GitHub comment post retry attempts |
| `SCOPE_SAGE_CONTEXT_MAX_TOKENS` | `150000` | Maximum token budget for `LlmContext` |
| `SCOPE_SAGE_LOG_LEVEL` | `info` | Tracing log level (`trace`, `debug`, `info`, `warn`, `error`) |
| `SCOPE_SAGE_DEPLOYMENT_ENV` | `production` | Deployment environment label used as the `deployment.environment` OTEL resource attribute |

### DocumentSource Format

`SCOPE_SAGE_DOCUMENT_SOURCES` is a JSON array:

```json
[
  {
    "label": "architecture-docs",
    "repo_url": "https://git.internal/org/architecture",
    "branch": "main",
    "include_globs": ["docs/**/*.md", "adr/**/*.md"]
  },
  {
    "label": "roadmap",
    "repo_url": "https://git.internal/org/roadmap",
    "branch": "main",
    "include_globs": ["*.md"]
  }
]
```

Each entry has a human-readable `label` used in audit records and failure notices when a source is unavailable.

---

## Networking

| Direction | Endpoint | Protocol |
|---|---|---|
| Inbound | GitHub webhook delivery | HTTPS (TLS terminated at load balancer or ingress) |
| Outbound | GitHub REST API (`api.github.com`) | HTTPS |
| Outbound | Anthropic API (`api.anthropic.com`) | HTTPS |
| Outbound | Internal documentation repositories | HTTPS |
| Outbound | OTEL exporter | gRPC or HTTP |
| Outbound | Alert target | HTTPS or SMTP |

The service does not initiate any connections except those listed above.

---

## GitHub App Configuration

When using `GithubWebhookSource`, the GitHub App must be configured in the GitHub App settings before deployment.

### Required Permissions

| Permission | Level | Purpose |
|---|---|---|
| `Issues` | Read & Write | Fetch issue content; post assessment and failure notice comments |

> Note: GitHub App `Issues: Write` grants comment creation. It does **not** grant label write access unless explicitly added. Do not add label permissions.

### Webhook Subscriptions

The GitHub App must subscribe to:

| Event | Action filter | Purpose |
|---|---|---|
| `Issues` | `labeled` | Triggers assessment when `state:design` is applied |

No other event subscriptions are required. Subscribe to `labeled` only — do not subscribe to all issue events.

### Webhook Endpoint

Configure the GitHub App webhook URL to: `https://<your-domain>/github/webhook`

Set the webhook secret to the same value as `SCOPE_SAGE_WEBHOOK_SECRET`.

TLS must be terminated externally (load balancer or ingress). The service endpoint (`/github/webhook`) responds only to `POST` requests.

---

## Health Checks

| Path | Method | Response |
|---|---|---|
| `/health/live` | GET | HTTP 200 `{"status":"ok"}` — process is running |
| `/health/ready` | GET | HTTP 200 `{"status":"ready"}` or HTTP 503 — ready to process events: signing key loaded, GitHub token valid; for webhook source: HTTP listener is bound; for queue source: broker connection is established |

Readiness probe is called by the supervisor before routing traffic to the instance.

---

## Observability

### Structured Logging

All log output uses `tracing` with structured fields. No plain strings. Log level is configured via `SCOPE_SAGE_LOG_LEVEL`.

Key log events (at `info` level):

- Service startup with resolved configuration summary (no secret values)
- Event received: `source_type`, `event_type`, `action`, `issue_number`, `repo`
- Assessment cycle start: `issue_ref`
- Document loading complete: `sources_loaded`, `sources_failed`, `total_bytes`
- LLM call: `model`, `prompt_tokens`, `completion_tokens`, `latency_ms`
- Comment posted: `issue_ref`, `comment_id`, `latency_ms`
- Cycle complete: `issue_ref`, `outcome`, `total_latency_ms`

### OTEL Resource Attributes

All OTEL signals (metrics, traces) must include these standard resource attributes to enable correlation with other pipeline services (GateKeeper, CogWorks):

| Attribute | Value |
|---|---|
| `service.name` | `scope-sage` |
| `service.version` | Binary build metadata (e.g. git tag or commit SHA, injected at compile time) |
| `deployment.environment` | Configured via `SCOPE_SAGE_DEPLOYMENT_ENV` environment variable (e.g. `production`, `staging`) |

### Metrics (OTEL)

| Metric | Type | Description |
|---|---|---|
| `scope_sage.events.received` | Counter | Total events received from the configured `IssueEventSource`, labelled by `source_type` (`webhook` or `queue`) and `event_type` |
| `scope_sage.events.rejected` | Counter | Events rejected before emission: HMAC failures (webhook) or schema validation errors (queue), labelled by `source_type` and `reason` |
| `scope_sage.assessments.started` | Counter | Assessment cycles started |
| `scope_sage.assessments.completed` | Counter | Assessment cycles completed, labelled by `outcome` |
| `scope_sage.assessments.duration_ms` | Histogram | End-to-end assessment cycle duration |
| `scope_sage.llm.calls` | Counter | LLM API calls, labelled by `model` and `outcome` |
| `scope_sage.llm.latency_ms` | Histogram | LLM API call latency |
| `scope_sage.llm.prompt_tokens` | Histogram | Prompt token count per call |
| `scope_sage.llm.completion_tokens` | Histogram | Completion token count per call |
| `scope_sage.github.api.calls` | Counter | GitHub API calls, labelled by `operation` and `outcome` |

### Tracing

Every assessment cycle is wrapped in a root `tracing` span: `scope_sage.assessment`. All child operations (document loading, LLM call, comment post) create child spans with relevant attributes.

---

## Alerting

Failure alerts are sent via the configured `ConfigurableFailureNotifier` when:

- An assessment cycle fails to complete (LLM error, all sources unavailable)
- Comment posting fails after all retries
- The service fails to start (logged to stderr — supervisor must forward to alerting)

Alert payload includes: `IssueRef`, UTC timestamp, failure reason, audit record ID.

---

## Key Management

See [security.md](security.md#key-management) for signing key rotation procedure.

The Ed25519 public key must be distributed to:

- GateKeeper (for signature verification before state transition)
- CogWorks (for signature verification before implementation start)

Update the public key in all downstream systems before revoking the old private key.

---

## Backup and Recovery

Scope Sage is stateless. There is nothing to back up. On failure:

1. Restart the service (supervisor handles automatically)
2. Re-apply the `state:design` label to re-trigger assessment (if an in-progress cycle was lost)
3. If the signing key was lost, generate a new key pair and distribute the new public key before restarting

---

## Runbook: Manual Retry

If Scope Sage posted a `FailureNotice` and the issue remains in `state:design`:

1. Check the audit log for the `IssueRef` to identify the failure reason
2. Resolve the root cause (e.g. restore document source access, check API key validity)
3. Remove the `state:design` label from the issue
4. Re-apply the `state:design` label — this triggers a fresh assessment cycle
5. The idempotency check will detect the `FailureNotice` comment is not an assessment comment and proceed normally

If the reviewer chooses to proceed without the document, they apply `state:implementation` directly (bypassing Scope Sage). GateKeeper enforces that this transition is human-only.

> **Queue deployments**: Re-applying the `state:design` label triggers a fresh assessment cycle only if the upstream queue publisher is configured to emit a message on label application. Verify with the queue operator that re-labelling will produce a new event before following this runbook. If not, request that the operator manually publish a `LabelEvent` for the affected issue.
