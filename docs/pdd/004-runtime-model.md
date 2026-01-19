# Runtime Model

This document focuses on the runtime mechanics that keep execution consistent and reliable across transports. Names and APIs are illustrative.

## Command Execution Lifecycle

When executing or processing a command with a `target`:

1. Validate input via schema.
2. Resolve target entity id + creation policy (from the handler).
3. Load/create entity instance (tracked) when a target is specified.
4. Run invariants scoped with `when(command)` before the handler.
5. Run handler in a staged execution context (events buffered).
6. Run invariants without `when` at the end of execution.
7. Persist entity + publish buffered events (only if all checks pass).

## Async Context and Transactions

Async context will make request-scoped tracing and transactions available without threading `ctx` everywhere. The framework will own the begin/commit/rollback lifecycle so application code only expresses intent. Handlers and services will retrieve context with `useContext()` instead of receiving it as a parameter.

Typical request-scoped capabilities exposed by `useContext()` should include auth/identity, logging/metrics, feature flags/config, repositories/adapters, command/event IO, etc.

```ts
await issueCardHandler.execute(input)
// framework handles tx begin/commit/rollback and ALS scope (execute stays local)
```

If explicit scoping is needed (tests, background jobs), a helper can establish context:

```ts
await withContext(testContext, async () => {
	await issueCardHandler.execute(input) // local execution in tests
})
```

## Temporal Decoupling

Application services choose timing:

- `commandHandler.execute(input)` runs the command now.
- `commandHandler.publish(input)` hands off to messaging infrastructure for distributed execution (future concern).

## Creation Policy Boundaries

Creation policies include `always`, `never`, and `if_missing`. They are defined on handlers so entity instantiation rules stay in the domain layer.

## One Entity Write per Command Execution

A command handler is bound to a single entity instance. Cross-entity workflows are expressed through events, event handlers, and additional commands over time.

Any future allowance for multi-entity writes would require a deliberate ADR.

## Outbox Pattern

Outbox support could provide a reliable way to persist events alongside state changes and publish them asynchronously, ensuring delivery even if the process crashes after a commit.

## Strategy Policies

Policies centralize strategy selection without scattering conditionals across handlers.

```ts
export const FraudAssessment = defineStrategy().name('FraudAssessment')

export const basicFraud = defineStrategyVariant(FraudAssessment)
  .name('basic')
  .assess(async () => 'allow')

export const fraudPolicy = defineStrategyPolicy()
  .strategy(FraudAssessment)
  .select((ctx) => (ctx.tenantId === 'enterprise' ? 'vendorX' : 'basic'))
```

## Auth Helpers

Authentication wrappers (for example, `withAuth` or `requireUser`) could be offered as opinionated utilities, but they are not required by the core framework.

## Repository Access

Repositories should provide explicit load/save access while keeping transaction boundaries and identity resolution inside the framework. Application services can use repositories to fetch context before orchestrating command execution so handlers stay focused on domain decisions. More adapter details live in [`docs/pdd/006-infrastructure.md`](006-infrastructure.md).

```ts
export const redeemCardService = defineApplicationService()
  .input(RedeemCard.schema)
  .handle(async (command) => {
    const giftCardRepository = useRepository(GiftCardRepository)
    const card = await giftCardRepository.load(command.cardId)

    if (!card) return { accepted: false }

    await redeemCardHandler.execute(command)

    return { accepted: true }
  })
```

