# System Overview

## Purpose

Scope Sage performs architectural pre-work on GitHub issues that have passed triage. Given an issue labelled `state:design`, it reads internal documentation and synthesises a structured architecture assessment, posted as an issue comment. It does not make decisions — it produces a reference document for a human reviewer.

## Position in the Pipeline

```
Triage Titan   automated classification and initial triage
     ↓
Human gate     reviews triage output, consciously decides to proceed
  applies → state:design
     ↓
Scope Sage     reads internal docs + synthesises architecture assessment
  posts structured comment on issue
     ↓
Human gate     reviews assessment, decides whether to proceed
  applies → state:implementation
     ↓
CogWorks       implements the work
```

The `state:design` label is **human-only**. A human reviews the Triage Titan output and makes a conscious decision before Scope Sage activates. This is a deliberate security gate — Scope Sage never operates on unreviewed input.

## What Scope Sage Does

Given an issue in `state:design`, Scope Sage executes the following sequence:

1. Receives a `LabelEvent` from the configured `IssueEventSource` (either `GithubWebhookSource` or `QueueEventSource`).
2. Validates the event: the `IssueEventSource` implementation handles transport-level authentication (HMAC for webhooks; broker auth for queues) before emitting the event.
3. Filters the event: confirms the label is `state:design` on an issue (not a PR).
4. Checks idempotency: if the issue already has a Scope Sage assessment comment, stops.
5. Reads the issue title, body, and metadata from the GitHub API.
6. Loads internal documentation from configured sources (architecture docs, ADR/RFC catalogue, roadmap).
7. Assembles the issue content and documentation into a structured LLM prompt.
8. Calls the LLM and parses the structured assessment response.
9. Renders the LLM output into the standard six-section assessment document.
10. Computes `SHA-256(title + "\n" + body)` and computes the `key-id` from the signing key.
11. Signs `"approved:<ts>\nkey-id:<key-id>\nhash:sha256:<hex>"` with the Ed25519 private key.
12. Posts the rendered document with the hidden signature block as an issue comment.
13. Records the entire operation in the audit log.

If any step from 5 onwards fails, Scope Sage posts a failure notice on the issue and alerts the configured reviewer. The issue remains in `state:design`.

## What Scope Sage Does Not Do

- Write or modify code in any repository
- Apply labels or transition issue state
- Make implementation decisions
- Interact with the issue author
- Approve or reject issues
- Operate on pull requests or branches

## Output Document Structure

Every assessment comment contains the following sections in order:

| Section | Contents |
|---|---|
| Strategic Alignment | Verdict (`aligned` / `misaligned` / `unclear`) with rationale referencing roadmap and OKRs |
| Affected Areas | Systems, services, modules, or interfaces likely touched |
| Architecture Fit | Consistency with target architecture; migration work or shims required |
| Relevant ADRs and RFCs | Links to governing decision records; explicit note if none exist |
| Cross-Cutting Concerns | Safety, interface, certification, or inter-service concerns |
| Open Questions | Unresolved architectural questions for the reviewer to answer |

The comment also contains a hidden signature block — an HTML comment not visible in the GitHub UI but parseable by downstream automation.

## System Context

```
                         ┌─────────────────────────────────────────┐
                         │              Scope Sage                 │
                         │                                         │
  GitHub Webhooks ──────▶│  GithubWebhookSource ┌───────────────┐  │
  (label events)         │   or QueueEventSource┘  │            │  │
  Message Queue ────────▶│            │              │            │  │
                         │            ▼              │            │  │
                         │  EventRouter              │            │  │
                         │       │                   │            │  │
                         │       ▼                   │            │  │
                         │  AssessmentOrchestrator   │            │  │
                         │    ├── IssueTracker ───────────────────┼──▶ GitHub API
                         │    ├── DocumentRepository ────────────┼──▶ Internal doc repos
                         │    ├── AssessmentEngine ──────────────┼──▶ Anthropic API
                         │    ├── DocumentRenderer (pure)          │
                         │    ├── DocumentSigner (pure)            │
                         │    ├── AuditLog ─────────────────────┤──▶ Audit sink
                         │    └── FailureNotifier ────────────────┼──▶ GitHub API + Alert sink
                         └─────────────────────────────────────────┘
```

## High-Level Data Flow

```
Label event received from IssueEventSource (webhook or queue)
  → transport-level authentication handled by the IssueEventSource implementation
  → filter: is label "state:design" on an issue?
  → check idempotency: existing Scope Sage comment?
  → fetch issue title, body, metadata from GitHub API
  → load internal documentation from configured sources
  → assemble LLM context (issue + documents + prompt template)
  → call LLM → parse structured six-section response
  → render assessment Markdown from template
  → compute SHA-256(title + "\n" + body), compute key-id from signing key
  → sign "approved:<ts>\nkey-id:<key-id>\nhash:sha256:<hex>" with Ed25519 private key
  → append hidden signature block to comment body
  → POST comment to GitHub issue
  → write audit record (LLM call + comment post + latencies)
```

## Glossary

See [vocabulary.md](vocabulary.md) for complete definitions of all domain terms used across this specification.
