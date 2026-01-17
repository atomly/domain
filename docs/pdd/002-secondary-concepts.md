# Secondary Framework Concepts

These ideas are adjacent to the current design and may become first-class concepts. Names and APIs are illustrative.

## Async Context and Transactions

Async context makes request-scoped tracing and transactions available without threading `ctx` everywhere. The framework owns the begin/commit/rollback lifecycle so application code only expresses intent. Handlers and services retrieve context with `useContext()` instead of receiving it as a parameter.

```ts
await ctx.commands.execute(IssueCard, input)
// framework handles tx begin/commit/rollback and ALS scope (execute stays local)
```

If explicit scoping is needed (tests, background jobs), a helper can establish context:

```ts
await withContext(testContext, async () => {
	await ctx.commands.execute(IssueCard, input) // local execution in tests
})
```

```ts
export const redeemCardHandler = defineCommandHandler({
	command: RedeemCard,
	handle: async function (cmd) {
		// no ctx parameter required; ALS provides request scope
		const giftCardRepository = useContext().giftCardRepository
		const card = await giftCardRepository.load(cmd.cardId)
		card.remainingValue -= cmd.amount
		await giftCardRepository.save(card)
	}
})
```

## Repositories

Repositories provide explicit load/save access while keeping transaction boundaries and identity resolution inside the framework. Handlers can focus on domain decisions without manual persistence plumbing.

```ts
export const redeemCardHandler = defineCommandHandler({
	command: RedeemCard,
	handle: async function (cmd) {
		const giftCardRepository = useContext().giftCardRepository
		const card = await giftCardRepository.load(cmd.cardId)
		card.remainingValue -= cmd.amount
		await giftCardRepository.save(card)
	}
})
```

## Outbox Pattern

Outbox support could provide a reliable way to persist events alongside state changes and publish them asynchronously, ensuring delivery even if the process crashes after a commit.

## Strategy Policies

Policies centralize strategy selection without scattering conditionals across handlers.

```ts
export const FraudAssessment = defineStrategy({ name: 'FraudAssessment' })

export const basicFraud = defineStrategyVariant(FraudAssessment, {
	name: 'basic',
	assess: async function () {
		return 'allow'
	}
})

export const fraudPolicy = defineStrategyPolicy({
	strategy: FraudAssessment,
	select: function (ctx) {
		return ctx.tenantId === 'enterprise' ? 'vendorX' : 'basic'
	}
})
```

## Auth Helpers

Authentication wrappers (for example, `withAuth` or `requireUser`) could be offered as opinionated utilities, but they are not required by the core framework.
