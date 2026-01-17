# AGENTS.md

This file provides guidance for agentic coding assistants working in this repo.
It captures build/test commands, coding conventions, and repo-specific context.
If these instructions conflict with system/user instructions, follow those.

## Repository overview

- This repo currently contains design documents only.
- There is no application source code or tooling config yet.
- The docs in `docs/` define the intended domain-first framework.

## Tooling status

There are no build, lint, or test tools defined in the repo yet.
If you add tooling in the future, update this file accordingly.

### Build commands

- Not defined. There is no build configuration in the repo.
- If a build system is introduced, document the primary command here.

### Lint/format commands

- Not defined. There is no lint or formatter configuration in the repo.
- If a linter/formatter is introduced, document the primary command here.

### Test commands

- Not defined. There is no test runner configuration in the repo.
- If a test framework is introduced, document the primary command here.

### Single-test commands

- Not defined. There is no test runner configuration in the repo.
- When a test framework is introduced, include examples for:
      - Running one test file
      - Running a single test by name
      - Running a focused subset (e.g., tag, grep, or pattern)

## Code style guidelines

There is no implementation code yet, so follow these general conventions
until the codebase provides more specific guidance.

### Language and module style

- Default to TypeScript for new code unless stated otherwise.
- Prefer ES module syntax (`import`/`export`) over `require`/`module.exports`.
- Keep files small and focused; avoid multi-concern modules.

### Imports

- Group imports by source:
    1) Node built-ins
    2) External dependencies
    3) Internal modules
- Separate groups with a single blank line.
- Use named imports over default when the library supports it.
- Avoid deep relative import chains; prefer scoped internal paths when added.

### Formatting

- Use 2-space indentation unless the repo introduces a formatter.
- Prefer single quotes in JS/TS unless escaping would be excessive.
- Keep lines reasonably short (around 100 characters).
- Avoid trailing whitespace and keep a newline at EOF.

### Types

- Use explicit types for public APIs and exported functions.
- Prefer type aliases for data shapes and interfaces for contracts.
- Avoid `any`; use `unknown` and narrow when needed.
- Model domain concepts with strong types instead of primitives.

### Naming conventions

- Use `camelCase` for variables/functions.
- Use `PascalCase` for types, classes, and constructors.
- Use `SCREAMING_SNAKE_CASE` for constants only when truly global.
- Names should reflect domain language from the docs.

### Error handling

- Validate input at the boundary (schema validation or equivalents).
- Prefer returning structured errors over throwing for expected failures.
- Throw only for programmer errors or truly exceptional conditions.
- Include context (domain identifiers, command names) in error payloads.

### Asynchronous behavior

- Use `async`/`await` for readability.
- Avoid unhandled promise chains; always return or await.
- Prefer explicit command enqueue/execute semantics for domain actions.

### Logging and observability

- Use structured logging with consistent fields when introduced.
- Propagate trace context when the tracing stack is added.
- Avoid logging sensitive data.

### Domain modeling guidance

- Encode domain language directly in names (commands, events, queries).
- Keep invariants close to the domain model.
- Separate application services (edge orchestration) from domain logic.
- Use events to express facts, commands to express intent, queries to ask.

## Documentation conventions

- Keep docs concise and narrative.
- Prefer examples that reflect the contents in `docs/`.

## Cursor/Copilot rules

- No `.cursor/rules/`, `.cursorrules`, or `.github/copilot-instructions.md`
  files exist in this repo currently.
- If these are added later, include their requirements here.

## When making changes

- Make minimal, focused edits.
- Do not introduce new files unless required.
- Keep new content aligned with the domain-first principles in `docs/`.
