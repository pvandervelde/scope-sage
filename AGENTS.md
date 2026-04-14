# AI Agent Guidelines

This file provides guidance and context for AI coding assistants working on this project.

## Project Overview

Scope Sage is a GitHub bot that performs architectural pre-work on issues that have passed triage. It sits between Triage Titan and CogWorks in the issue pipeline, activated only after a human has reviewed the triage output and decided the issue is worth progressing. Its output is a structured architecture document posted as a comment on the issue, giving the reviewing engineer — and subsequently CogWorks — a clear picture of how the proposed work fits within the current and target architecture.

Scope Sage does not make decisions. It produces a reference document for human review. A human gate follows its output before any implementation work begins.

Key traits:
- **Read-synthesise-report**: Scope Sage reads internal documentation (architecture docs, ADR/RFC catalogue, roadmap) and synthesises an architecture assessment using an LLM. It never writes code or modifies repository content.
- **Human-gated trigger**: Activated only by the `state:design` label, which is applied exclusively by humans, ensuring it never operates on unreviewed input.
- **Minimal write surface**: Write access is limited to issue comments only. No label transitions, no PR operations, no code writes.
- **Cryptographic output signing**: Every posted document includes a hidden Ed25519 signature block so downstream automation can verify the document's authenticity without Scope Sage needing elevated permissions.
- **Audit trail**: Every LLM call and document posting is recorded for post-hoc review.

See [doc/specs/overview.md](doc/specs/overview.md) for the full system context and [doc/specs/README.md](doc/specs/README.md) for the spec index.

## Production Software Standards

**This is production-grade software.** All code must meet production quality standards:

- **Complete Implementation**: No TODOs, placeholders, or "demonstration" code. Every feature must be fully implemented.
- **Comprehensive Error Handling**: All error paths must be handled properly, with clear error messages and proper error propagation.
- **Full Test Coverage**: All functionality must have comprehensive tests covering happy paths, error cases, and edge conditions.
- **Production-Ready Documentation**: All public APIs must have complete rustdoc with examples, error conditions, and behavioral specifications.
- **Security First**: All security-sensitive operations must be implemented with production-grade security measures.
- **Performance Conscious**: Code must be optimized for production workloads, not just correctness.
- **Observability**: All operations must have appropriate logging, metrics, and tracing for production debugging.

When implementing features:

- Write production code from the start - no prototypes or demos
- Think about failure scenarios and edge cases
- Consider operational concerns (monitoring, debugging, maintenance)
- Implement complete functionality, not partial demonstrations

## Pre-Implementation Checklist

Before implementing features, verify:

1. **Read Specifications**: Check doc/specs/ for relevant documentation
2. Read [doc/specs/constraints.md](doc/specs/constraints.md) (implementation rules and tripwires)
3. Read [doc/specs/requirements.md](doc/specs/requirements.md) (what must be true — use to confirm scope before adding code)
4. Read relevant standards in [doc/standards/](doc/standards/) (language/domain specific; may be empty early in the project)
5. **Search Existing Code**: Use semantic_search to find similar implementations
6. **Check Module Structure**: Determine if code belongs in existing module or needs new one
7. **Security Review**: Identify sensitive data (tokens, secrets) requiring special handling
8. **Plan Tests**: Identify test scenarios before writing implementation

## Summary

Following these conventions ensures:

- **Consistency**: Codebase looks like one person wrote it
- **Maintainability**: Easy to find and understand code
- **Quality**: High test coverage and clear documentation
- **Security**: Sensitive data handled properly
- **Performance**: Conscious resource management

When in doubt, look at existing code in the repository as examples of these patterns in practice.

## Workflow

1. Check for existing ADRs related to your task
2. Follow coding standards in .tech-decisions.yml
3. Write tests before implementation (TDD preferred)
4. Run pre-commit hooks before committing
5. Include context in commit messages

## Task Management

This project uses Beads (bd) for AI-friendly task tracking.

### Before starting work

1. Check what's ready: `bd ready --json`
2. Pick a task: `bd show bd-abc --json`
3. Start work: `bd update bd-abc working`

### When creating new tasks

1. Create issue: `bd create "Task description" -p 1 -t feature`
2. Add dependencies: `bd update bd-xyz --blocks bd-abc`
3. The task will auto-appear in `bd ready` when blockers are done

### When finishing work

1. Commit with issue ID: `git commit -m "Fix auth bug (bd-abc)"`
2. Close issue: `bd close bd-abc --reason "Completed"`
3. Sync: `bd sync` (usually automatic)

### Integration with ADRs

- Link ADRs in task descriptions: "See ADR-0005 for context"
- Create tasks for implementing ADR decisions
- Reference task IDs in ADR implementation notes

### Quick reference

``bash
bd ready              # Show tasks ready to work on
bd create "desc" -p 1 # Create new task (priority 1-5)
bd show bd-xyz        # Show task details
bd update bd-xyz working  # Mark task in progress
bd close bd-xyz       # Close completed task
bd search "keyword"   # Search tasks
bd doctor             # Check for orphaned work
``
