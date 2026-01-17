# Primary Concepts

## Concepts and Definition APIs

The framework concept set is organized around explicit definition APIs that preserve language, boundaries, and constraints. Concepts are declared independently to keep intent clear.

* `defineApplicationService`: edge orchestration and timing
* `defineCommand`: intent to change state
* `defineCommandHandler`: domain decision point for commands
* `defineEntity`: domain state and identity (use `parent` for child entities)
* `defineInvariants`: business rules around state transitions
* `defineEvent`: change that occurred
* `defineEventHandler`: reaction to domain events
* `defineQuery`: question about the system
* `defineQueryHandler`: read model access
* `defineDomainService`: multi-read domain logic with constraints

Schemas will be co-located with the concept definition. Examples use `Zod` for readability; the framework will accept any `Standard Schema`-compatible validator under the hood.

These APIs are illustrative and may change as ADRs refine naming, validation, and handler structure.

These decisions should stay aligned with DDD principles and the vision’s focus on explicit language, boundaries, and invariant preservation.

## Lifecycle and Deterministic Rules (Exploratory)

### Command Execution Lifecycle

When executing or processing a command with a `target`:

1. Validate input via schema.
2. Resolve target entity id + creation policy (from the handler).
3. Load/create entity instance (tracked) when a target is specified.
4. Run invariants:
   * entity-level `before`, then command-level `before`.
5. Run handler in a staged execution context (events buffered).
6. Run invariants:
   * command-level `after`, then entity-level `after`.
7. Persist entity + publish buffered events (only if all checks pass).

Creation policies include `always`, `never`, and `if_missing`.

### Temporal Decoupling

Application services choose timing:

* `execute(command, input)` runs the command now.
* `enqueue(command, input)` accepts now and runs later.

### Execution Context and Transactions

Execution runs inside request-scoped context backed by `AsyncLocalStorage`. The framework uses this context to manage tracing, transactions, and persistence lifecycles consistently:

* start a transaction when command execution begins
* commit when all invariants pass and persistence succeeds
* roll back on errors or failed invariants
* expose request-scoped utilities (tracing, auth, ids) without manual plumbing

Testing utilities can create a scoped context to make handlers deterministic without requiring explicit `ctx` wiring.

### One Entity Write per Command Execution

A command handler is bound to a single entity instance. Cross-entity workflows are expressed through events, event handlers, and additional commands over time.

Any future allowance for multi-entity writes would require a deliberate ADR.

## Traceability and Observability (Exploratory)

Traceability is shown here as a framework-owned default and may be scoped differently in ADRs. Application code does not manage tracing; the framework handles propagation, spans, and export end to end.

### Tracing Architecture

The framework owns trace propagation and span creation across the execution pipeline:

* **Edge adapters** extract and inject W3C trace context for HTTP, queues, and event handlers.
* **Execution pipeline** wraps application services, command execution, event handling, and query handling with spans.
* **Message correlation** attaches trace identifiers to commands/events/queries to preserve causality across boundaries.
* **Export layer** forwards traces to OTLP/OpenTelemetry destinations via a pluggable exporter.

### Standards

* **W3C Trace Context** for propagation (`traceparent`, `tracestate`)
* **OpenTelemetry** for instrumentation APIs and semantic conventions
* **OTLP** as the primary export protocol for traces/metrics/logs

### Framework Responsibilities

The framework will:

* extract and propagate W3C trace context across HTTP and message transports
* create spans for key lifecycle phases (application services, command execution, event handling)
* export telemetry via OTLP (configurable)

Application code will not manually construct tracing metadata.

## Boundaries

* Each boundary owns its language and models.
* Boundaries communicate via commands/events, not direct calls into another boundary’s domain.
* Event handlers live in the receiving boundary and translate inbound facts into local actions.

### Note: Domain Events vs Integration Events (Future Consideration)

We may introduce a distinction between:

* **Domain events**: internal facts in a boundary’s ubiquitous language
* **Integration events**: stable, cross-boundary contracts with explicit versioning/compatibility expectations

If adopted, cross-boundary communication would prefer integration events, while domain events remain internal. This decision is intentionally deferred to ADRs.

## Future Considerations

These items should feed ADRs for decisions that are still unsettled.

* Naming and introspection: exported identifier inference vs explicit names.
* Creation policy placement and semantics (handler-scoped `always`/`never`/`if_missing`).
* Entity state schemas: required vs optional and how they integrate with persistence.
* Integration events vs domain events for cross-context boundaries.
* Traceability defaults and which spans are mandatory vs configurable.
