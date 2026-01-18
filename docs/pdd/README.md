# Preliminary Design Document

## Purpose and Status

This document is a **brainstorming and sanity-check artifact** that explores how the vision in `docs/VISION.md` could translate into framework concepts. It is intentionally **non-binding** and should be read as a starting reference rather than a committed design. The language uses future tense for readability, but nothing here is finalized.

The goal is a **candidate application-facing model** for discussion, focused on public concepts and lifecycle guarantees instead of implementation details. The framing is DDD-inspired and centered on explicit domain language.

## Alignment with Vision

The document takes the vision’s principles as guardrails:

* domain language is encoded directly in code
* boundaries and invariants remain explicit
* temporal decoupling is a first-class concern
* traceability stays visible across time

## Sections

- [`docs/pdd/000-primary-concepts.md`](000-primary-concepts.md)
- [`docs/pdd/001-user-stories.md`](001-user-stories.md)
- [`docs/pdd/002-secondary-concepts.md`](002-secondary-concepts.md)
- [`docs/pdd/003-infrastructure.md`](003-infrastructure.md)
