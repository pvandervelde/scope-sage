# ADR-0006: Read-Only Access to Internal Documentation

Status: Accepted
Date: 2026-04-15
Owners: scope-sage

## Context

Scope Sage needs to read internal documentation (architecture docs, ADR/RFC catalogue, roadmap) to provide context to the LLM. It also needs to post comments on GitHub issues. The question is whether Scope Sage should have write access beyond issue comments — specifically, whether it should have access to label writes, PR operations, or any repository content writes.

The principle of least privilege requires that Scope Sage be granted only the permissions it needs to function and no more. Every additional permission is an additional blast radius in the event of a token compromise.

## Decision

Scope Sage's write surface is limited to **GitHub issue comments only**. It has no write access to:

- Labels (no label add, remove, or create)
- Pull requests (no PR creation, review, or merge)
- Repository content (no commits, pushes, or file writes)
- Issue state (no open/close operations)

Internal documentation is accessed **read-only** from a fixed, configured set of sources (`SCOPE_SAGE_DOCUMENT_SOURCES`). No source URLs may be dynamically generated from issue content.

## Consequences

- **Enables**: minimal blast radius on token compromise; no risk of Scope Sage accidentally or maliciously transitioning issue state; clear audit trail (the only Scope Sage writes are issue comments identified by the bot account).
- **Forbids**: Scope Sage using labels as bookkeeping state (e.g., "processing" or "assessed" labels); Scope Sage triggering any downstream automation by writing to repositories.
- **Trade-offs accepted**: Scope Sage cannot communicate state through labels. Monitoring must rely on the audit log and the presence of the assessment comment. The failure path must post a comment rather than setting a label, since comment posting is the only write operation available.

## Alternatives Considered

- **Write access to labels**: would allow Scope Sage to apply a "processed" label for bookkeeping. Rejected — labels are a state machine controlled by humans and GateKeeper. Scope Sage's involvement in the label graph creates coupling and ambiguity about who controls state transitions.

- **Write access to repository files**: would allow Scope Sage to save its assessment as a file in a designated folder. Rejected — unnecessary write surface; the assessment comment on the issue is the appropriate and auditable location.

- **Dynamic document source resolution from issue content**: would allow issue authors to specify additional documentation to include. Rejected — would allow SSRF (an issue could reference an internal metadata endpoint) and prompt injection (controlled document content). Sources are fixed at configuration time.

## Implementation Notes

- GitHub App permissions required:
  - `issues: read` — fetch issue content
  - `issues: write` — post comments (note: GitHub App permission `issues:write` grants comment creation but does not grant label write without explicit `issues:write` for labels — confirm minimum required permission scope)
  - Repository content access: no `contents` permission required (documentation is accessed via `DocumentRepositoryPort`, not through the GitHub issues API)
- The GitHub App installation configuration must be audited to ensure no permissions beyond those above are granted.
- `DocumentRepository` implementation accesses git repositories using a dedicated read-only credential (separate from the GitHub App token used for issues API).

## References

- [overview.md](../specs/overview.md#what-scope-sage-does-not-do)
- [security.md](../specs/security.md#threat-04-ssrf-via-document-source-configuration) — THREAT-04
- [responsibilities.md](../specs/responsibilities.md#gitdocumentrepository-implements-documentrepository)
