# Open Questions

These items should feed discussions for decisions that are still unsettled.

## Core Concepts

- Naming and introspection: exported identifier inference vs explicit names.
- Creation policy placement and semantics (handler-scoped `always`/`never`/`if_missing`).
- Entity state schemas: required vs optional and how they integrate with persistence.
- Integration events vs domain events for cross-context boundaries.
- Traceability defaults and which spans are mandatory vs configurable.
- Command `target` without `.entity(...)`: allow as stateless routing or treat as config error.
- Application service auto-load: allow `.loadEntity(...)` helper or keep manual repository access.

## AI Integrations

- Tool approval workflows before executing commands.
- MCP tools (Model Context Protocol) as adapters.
- Per-tenant model routing and quotas.
- Structured output validation with Zod/JSON schema before command execution.

## Infrastructure

- Configuration strategy for infrastructure concerns (database URLs, API keys, observability exporters).
- Command auto-registration inputs and expected DX.
- IoC auto-registration vs explicit adapter registration at boot.
- Requirements for auto-registration (manifest generation, stable naming, CLI tooling).
- Environment profiles for adapter selection and overrides.
