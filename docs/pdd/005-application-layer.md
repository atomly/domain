# Application Layer

Application services translate transport concerns into domain inputs and pick execution timing. They generate new identifiers before executing or publishing commands, depending on whether the workflow runs locally or through messaging infrastructure. We use `defineApplicationService` to keep edge logic separate from domain decisions. Authentication helpers like `requireUser()` are optional conveniences, not core framework concepts.

Application services *should* not:

* handle business logic
* raise domain events

They *should*:

* handle application concerns (auth, mapping, response shaping)
* orchestrate command processing timeline: execute (immediate) vs. publish (messaging/distributed)
* *sometimes* manage persistence

```ts
import crypto from 'node:crypto'

export const issueCardService = defineApplicationService()
  .input(
    z.object({
      amount: z.number().int().positive()
    })
  )
  .handle(async (input) => {
    await requireUser()

    const cardId = crypto.randomUUID()

    await issueCardHandler.execute({ cardId, amount: input.amount })

    return { cardId }
  })
```

## Service Composition Patterns

Applications may prefer single-use-case services or group multiple use cases together for a more controller-friendly DX. All options below preserve domain-first command/event boundaries.

### Router-style Service

```ts
export const userService = {
  create: defineApplicationService()
    .input(CreateUserInput)
    .handle(async (input) => {
      await createUserHandler.execute(input)
      return { ok: true }
    }),
  deactivate: defineApplicationService()
    .input(DeactivateUserInput)
    .handle(async (input) => {
      await deactivateUserHandler.execute(input)
      return { ok: true }
    })
}
```

### Use-case Namespace

```ts
export const userService = defineApplicationService({
  create: defineUseCase()
    .input(CreateUserInput)
    .handle(async (input) => {
      await createUserHandler.execute(input)
      return { ok: true }
    }),
  deactivate: defineUseCase()
    .input(DeactivateUserInput)
    .handle(async (input) => {
      await deactivateUserHandler.execute(input)
      return { ok: true }
    })
})
```

### Single-use-case Services + Router Composition

```ts
export const createUserService = defineApplicationService()
  .input(CreateUserInput)
  .handle(async (input) => {
    await createUserHandler.execute(input)
    return { ok: true }
  })

export const deactivateUserService = defineApplicationService()
  .input(DeactivateUserInput)
  .handle(async (input) => {
    await deactivateUserHandler.execute(input)
    return { ok: true }
  })

export const userRouter = defineServiceRouter({
  create: createUserService,
  deactivate: deactivateUserService
})
```
