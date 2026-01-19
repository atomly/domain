# Core Concepts

## Concepts and Definition APIs

The framework concept set is organized around explicit definition APIs that preserve language, boundaries, and constraints. Concepts are declared independently to keep intent clear.

* `defineApplicationService`: edge orchestration and timing
* `defineCommand`: intent to change state
* `defineCommandHandler`: domain decision point for commands
* `defineEntity`: domain state and identity
* `defineInvariants`: business rules around state transitions
* `defineEvent`: change that occurred
* `defineEventHandler`: reaction to domain events
* `defineQuery`: question about the system
* `defineQueryHandler`: read model access
* `defineDomainService`: multi-read domain logic with constraints

Schemas will be co-located with the concept definition. These APIs are illustrative and may change as we refine naming, validation, and handler structure.

These decisions should stay aligned with DDD principles and the vision’s focus on explicit language, boundaries, and invariant preservation.

## Examples

These snippets show the shape of each definition API. Examples use `Zod` for readability; the framework will accept any `Standard Schema`-compatible validator under the hood.

### Application Service

```ts
export const issueCardService = defineApplicationService()
  .input(
    z.object({
      cardId: z.string().min(1),
      amount: z.number().int().positive()
    })
  )
  .handle(async (input) => issueCardHandler.execute(input))
```

### Command

```ts
export const IssueCard = defineCommand()
  .schema(
    z.object({
      cardId: z.string().min(1),
      amount: z.number().int().positive()
    })
  )
  .target((command) => command.cardId)
```

### Command Handler

```ts
export const issueCardHandler = defineCommandHandler()
  .entity(GiftCard)
  .command(IssueCard)
  .creation('always')
  .handle((command, state) => {
    state.id = command.cardId
    state.remainingValue = command.amount

    raise(CardIssued, {
      cardId: command.cardId,
      amount: command.amount
    })
  })
```

Entity-bound handlers require a command with a `target` function. Commands without a target cannot bind to an entity; the handler should be configured without `.entity(...)` and manage repository access manually. A command with a `target` but no `.entity(...)` on the handler is an open design choice (stateless handler vs configuration error).

### Entity

```ts
export const GiftCard = defineEntity()
  .schema(
    z.object({
      id: z.string().min(1),
      remainingValue: z.number().int().nonnegative()
    })
  )
  .id((state) => state.id)
```

#### Class-based Entity API

Entities can remain simple classes while command handlers are expressed as composable subclasses. The handler class owns the command metadata and the `handle` method, and composing it into the entity binds the handler to that entity instance (so `this` is the state).

```ts
type CommandType<T> = T extends CommandDefinition<infer Input> ? Input : never

abstract class Entity<TId = string> {
  abstract id: TId
  equals(other: Entity<TId>) {
    return this.constructor === other.constructor && this.id === other.id
  }
}

class BaseModel extends Entity {
  constructor(
    public id: string,
    public remainingValue: number
  ) {}
}

class RedeemCardHandler extends CommandHandler<BaseModel>({
  command: RedeemCard,
  creation: 'never'
}) {
  handle(command: CommandType<typeof RedeemCard>) {
    this.balance -= command.amount
    raise(CardRedeemed, { cardId: command.cardId, amount: command.amount })
  }
}

class GiftCard extends compose(BaseModel, RedeemCardHandler) {}

withContext(() => {
  const giftCard = new GiftCard('123', 100)
  giftCard.redeem(50)
})
```

### Invariants

```ts
export const giftCardRules = defineInvariants()
  .entity(GiftCard)
  .check([
    {
      code: 'GIFT_CARD_NEGATIVE_BALANCE',
      message: 'Gift card remaining value must never be negative',
      check: (state) => state.remainingValue >= 0
    }
  ])
```

### Event

```ts
export const CardIssued = defineEvent()
  .schema(
    z.object({
      cardId: z.string().min(1),
      amount: z.number().int().positive()
    })
  )
```

### Event Handler

```ts
export const onCardIssued = defineEventHandler()
  .on(CardIssued)
  .handle(async (event) => {
    await notifyCardIssuedHandler.publish({ cardId: event.cardId })
  })
```

### Query

```ts
export const GetGiftCardBalance = defineQuery()
  .schema(
    z.object({
      cardId: z.string().min(1)
    })
  )
  .result(
    z.object({
      cardId: z.string().min(1),
      remainingValue: z.number().int().nonnegative()
    })
  )
```

### Query Handler

```ts
export const getGiftCardBalance = defineQueryHandler()
  .query(GetGiftCardBalance)
  .handle(async (query) => {
    const readModel = useRepository(GiftCardReadModel)
    return readModel.getById(query.cardId)
  })
```

### Domain Service

```ts
export const assessRedemption = defineDomainService()
  .input(
    z.object({
      cardId: z.string().min(1),
      amount: z.number().int().positive()
    })
  )
  .handle(async (input) => {
    const readModel = useRepository(GiftCardReadModel)
    const card = await readModel.getById(input.cardId)
    return { allowed: Boolean(card && card.remainingValue >= input.amount) }
  })
```

## Runtime Lifecycle

The runtime lifecycle and execution semantics are described in [`docs/pdd/004-runtime-model.md`](004-runtime-model.md).

## Observability

Observability is a framework-owned concern that should stay consistent across transports and handlers. The detailed observability model lives in [`docs/pdd/008-observability.md`](008-observability.md).

## Boundaries

* Each boundary owns its language and models.
* Boundaries communicate via commands/events, not direct calls into another boundary’s domain.
* Event handlers live in the receiving boundary and translate inbound facts into local actions.

### Domain Events vs Integration Events

We may introduce a distinction between:

* **Domain events**: internal facts in a boundary’s ubiquitous language
* **Integration events**: stable, cross-boundary contracts with explicit versioning/compatibility expectations

If adopted, cross-boundary communication would prefer integration events, while domain events remain internal. This decision is intentionally deferred for the time being.
