# Repositories

Repositories are the persistence ports for entities and read models. They provide explicit load/save access while keeping transaction boundaries and identity resolution inside the framework.

## Responsibilities

- Load entity state by identity and return a tracked instance.
- Persist entity state after command execution.
- Expose read models for query handlers and application services.
- Integrate with the request-scoped transaction lifecycle.

## Repository Mapping Options

The framework needs to know which repository owns an entity when auto-loading or auto-saving. Two options are worth keeping in mind.

### Option A: Explicit Registry Mapping

Register a repository for each entity in a provider registry.

```ts
const registry = createProviderRegistry()

registry.registerRepository(GiftCard, new GiftCardRepository(pg))

const framework = createFramework({ providers: registry })
```

### Option B: Repository Declares Entity

A repository declares which entity it owns, and the framework builds the mapping at boot.

```ts
export abstract class EntityRepository<TState> {
  static entity: EntityDefinition<unknown>
}

export class GiftCardRepository extends EntityRepository<GiftCardState> {
  static entity = GiftCard

  async load(id: string) {
    /* ... */
    return null
  }

  async save(state: GiftCardState) {
    /* ... */
  }
}
```

Both approaches should be discussed before we choose the final direction.

## Entity Repositories

Entity repositories provide the persistence port for command handlers and application services.

```ts
export abstract class PostgresRepository<TState> extends EntityRepository<TState> {
  constructor(protected readonly tx: PgTransaction) {
    super()
  }

  protected get db() {
    return this.tx.context
  }
}

export class PostgresGiftCardRepository extends PostgresRepository<GiftCardState> {
  async load(id: string) {
    return this.db.query.giftCards.findFirst({ where: eq(giftCards.id, id) })
  }

  async save(state: GiftCardState) {
    await this.db.insert(giftCards).values(state).onConflictDoUpdate({
      target: giftCards.id,
      set: state
    })
  }
}
```

## Read Models

Read models power queries without touching write-side invariants. The read model could also be a model separate from the Entity itself, but that will be out of scope for this PDD, and likely v0 of the framework.

```ts
export const getGiftCardBalance = defineQueryHandler()
  .query(GetGiftCardBalance)
  .handle(async (query) => {
    const repository = useRepository(GiftCardRepository)
    return repository.getById(query.cardId)
  })
```

## Repository Access in Services

Application services can use repositories to fetch context before orchestrating command execution so handlers stay focused on domain decisions.

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
