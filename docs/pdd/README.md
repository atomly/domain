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

- [`docs/pdd/001-introduction.md`](001-introduction.md)
- [`docs/pdd/002-user-stories.md`](002-user-stories.md)
- [`docs/pdd/003-core-concepts.md`](003-core-concepts.md)
- [`docs/pdd/004-runtime-model.md`](004-runtime-model.md)
- [`docs/pdd/005-application-layer.md`](005-application-layer.md)
- [`docs/pdd/006-repositories.md`](006-repositories.md)
- [`docs/pdd/007-infrastructure.md`](007-infrastructure.md)
- [`docs/pdd/008-observability.md`](008-observability.md)
- [`docs/pdd/009-controllers.md`](009-controllers.md)
- [`docs/pdd/010-ai-integrations.md`](010-ai-integrations.md)
- [`docs/pdd/011-open-questions.md`](011-open-questions.md)
