# ADR-0002: Deploy as Long-Running Service, Not GitHub Action

Status: Accepted
Date: 2026-04-15
Owners: scope-sage

## Context

Scope Sage needs to:
1. Receive inbound HTTPS webhook events from GitHub.
2. Hold an Ed25519 private key in memory securely across multiple assessment cycles.
3. Access internal documentation repositories that are not publicly reachable.
4. Reuse connections to the GitHub API and Anthropic API for efficiency.

The two primary deployment options are a GitHub Action (ephemeral, triggered per event) and a long-running service on internal infrastructure.

## Decision

Deploy Scope Sage as a long-running service on internal infrastructure. It listens for GitHub webhook events on an HTTPS endpoint and processes each event as it arrives. It is not deployed as a GitHub Action.

## Consequences

- **Enables**: persistent inbound webhook receiver; access to non-public internal networks; stable in-memory key material; connection reuse to external APIs; consistent startup configuration validation.
- **Forbids**: using GitHub Actions workflow YAML to orchestrate assessment steps.
- **Trade-offs accepted**: infrastructure provisioning and management is required (process supervisor, health monitoring, TLS termination, networking). Operational complexity is higher than GitHub Actions.

## Alternatives Considered

- **GitHub Action**: triggered by `on: issues: types: [labeled]`. Cannot receive inbound webhooks directly; ephemeral environment cannot safely hold the signing private key; cannot reach internal documentation repositories without additional network bridging.
- **Serverless function (Lambda, Cloud Run)**: lower operational overhead than a persistent service, but cold start latency affects responsiveness; private key management in ephemeral compute is complex; outbound network access to internal repositories requires additional configuration that negates the simplicity advantage.

## Implementation Notes

- The Cargo binary entry point starts an `axum` HTTP server bound to the configured port.
- A process supervisor (systemd, Kubernetes Deployment) provides automatic restart on failure.
- TLS is terminated at the load balancer or ingress controller — the service listens on plain HTTP internally.
- The service exposes `/health/live` and `/health/ready` endpoints for supervisor probes.
- Graceful shutdown on SIGTERM: drain in-flight requests for up to 30 seconds, then exit.

## References

- [operations.md](../specs/operations.md) — deployment configuration and health checks
- [security.md](../specs/security.md) — key management requirements
