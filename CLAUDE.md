# CLAUDE.md

## Repository Purpose

This is a design reference library for modular software systems. It contains architectural documentation, design patterns, and concrete system designs — not application code. The primary output is Markdown documents with embedded code skeletons.

## Document Conventions

- All documents use Markdown with Mermaid diagrams for architecture visuals
- Code examples are real, runnable Python — not pseudocode
- Every document follows the structure defined in README.md (Context, Design Decisions, Architecture, etc.) but only includes sections that are relevant
- Documents in `patterns/` are technology-specific but domain-agnostic
- Documents in `systems/` compose patterns and add domain-specific context
- Documents in `principles/` are conceptual — focused on reasoning and tradeoffs, with minimal code examples where they clarify a design decision
- Cross-reference related documents using relative links: `[Kafka Topology](../patterns/messaging/kafka-topology.md)`

## Writing Style

- Lead with the problem, not the solution
- Always include tradeoffs — no pattern is universally correct
- Failure modes matter as much as happy paths
- Be specific: name actual libraries, actual config values, actual error scenarios
- Mermaid diagrams should be readable without surrounding text

## File Organization

- One concept per file — don't combine multiple patterns into one document
- Use kebab-case for file names: `sqlalchemy-repository.md`, not `SQLAlchemy_Repository.md`
- New system designs go in `systems/<system-name>/` with an `overview.md` as entry point
- New patterns go in `patterns/<category>/` — create a new category only if nothing existing fits
- New cross-cutting data architecture docs go in `data-strategies/`

## Code Skeletons

- Python 3.12+ with type hints
- Use Pydantic v2 for schemas and validation
- Use Python Protocols for module interfaces (not ABC)
- SQLAlchemy 2.0 style (mapped_column, not Column)
- Async-first where it matters (FastAPI routes, DB queries), sync where simpler
- Include dependency versions in code blocks when they matter
