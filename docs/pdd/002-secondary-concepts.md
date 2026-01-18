# Secondary Framework Concepts

These ideas are adjacent to the current design and may become first-class concepts. Names and APIs are illustrative.

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

## Repositories

Repositories should provide explicit load/save access while keeping transaction boundaries and identity resolution inside the framework. Application services can use repositories to fetch context before orchestrating command execution so handlers stay focused on domain decisions.

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
