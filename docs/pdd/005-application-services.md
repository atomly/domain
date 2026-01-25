# Application Services

Application services are the single application boundary. They translate transport inputs into domain intent, handle edge concerns (auth, validation, response shaping), and decide execution timing (`execute` vs `publish`). We use `defineApplicationService` to keep orchestration separate from domain decisions. Authentication helpers like `requireUser()` are optional conveniences, not core framework concepts.

Application services *should* not:

* handle business logic
* raise domain events

They *should*:

* handle application concerns (auth, mapping, response shaping)
* orchestrate command processing timeline: execute (immediate) vs. publish (messaging/distributed)
* initialize the ALS context for a use case
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

Application services should accept an optional `context` seed (trace IDs, auth, request metadata) so adapters can pass request-level data without owning the ALS lifecycle. For example, `issueCardService.handle(input, { context })`.

## Service Registry

Application services are registered as a plain object on the framework. Each key may be a single service or a group of services with a maximum depth of 1.

```ts
const giftCardServices = {
  issue: defineApplicationService()
    .input(z.object({ amount: z.number().int().positive() }))
    .handle(async (input, { context }) => {
      const cardId = context.ids.uuid()
      await issueCardHandler.execute({ cardId, amount: input.amount })
      return { cardId }
    }),
  redeem: defineApplicationService()
    .input(z.object({ cardId: z.string(), amount: z.number().int().positive() }))
    .handle(async (input) => {
      await redeemCardHandler.execute(input)
      return { ok: true }
    })
}

const services = {
  giftCard: giftCardServices,
  health: defineApplicationService().handle(async () => ({ ok: true }))
}

const framework = createFramework({ services })
```

Rules:

- Maximum depth is 1 (`giftCard.issue` is allowed, `giftCard.admin.freeze` is not).
- A key cannot be both a service and a group.
- Adapters can address services by key (for example, `giftCard.issue`).

## Transport Adapters

Transport adapters translate HTTP, RPC, queues, and CLI inputs into application service calls while handling protocol-specific concerns (routing, headers, status codes, acknowledgements). They are intentionally thin and should only seed request metadata (headers, auth, request IDs) into the service call.

### Transport Patterns

#### REST

```ts
const createUserHandler = defineCommandHandler()
  .entity(User)
  .command(CreateUser)
  .creation('always')
  .handle((command, state) => {
    state.id = command.id
    state.email = command.email
    raise(UserCreated, { id: command.id })
  })

const services = {
  users: {
    create: defineApplicationService()
      .input(CreateUserInput)
      .handle(async (input) => {
        await createUserHandler.execute(input)
        return { ok: true }
      })
  }
}

const framework = createFramework({ services })
const http = expressAdapter(framework, { app })

http.post('/users', framework.services.users.create)
```

#### GraphQL

```ts
const resolvers = {
  Query: {
    card: (_: unknown, args: { id: string }, ctx: unknown) =>
      getCardService.handle(args, { context: ctx })
  },
  Mutation: {
    issueCard: (_: unknown, args: IssueCardInput, ctx: unknown) =>
      issueCardService.handle(args, { context: ctx })
  }
}

// adapter helpers can expose a context seed function
// so GraphQL servers can pass request metadata
```

#### RPC (JSON-RPC / gRPC)

```ts
rpcServer.on('request', async (req) => {
  if (req.method === 'card.get') return getCardService.handle(req.params)
  if (req.method === 'card.issue') return issueCardService.handle(req.params)
  throw new RpcError('METHOD_NOT_FOUND')
})

// context seeding can be applied at the transport boundary
```

#### oRPC

```ts
const api = orpcAdapter(framework, { os })

const handler = new RPCHandler(api.router, {
  plugins: [new CORSPlugin()],
  interceptors: [onError(console.error)]
})
```

The adapter should seed context from oRPC headers and share the same OpenTelemetry SDK as the framework instrumentation.

#### tRPC

```ts
const api = trpcAdapter(framework, {
  t,
  createContext: ({ req }) => framework.seedContext({ headers: req.headers })
})

export const appRouter = api.router
export const handler = api.httpHandler()
```

#### WebSockets (Real-time Gateways)

```ts
wsServer.on('message', async (message) => {
  const input = JSON.parse(message.toString())
  const result = await issueCardService.handle(input)
  wsServer.send(JSON.stringify(result))
})
```

#### Webhooks (Inbound Integrations)

```ts
app.post('/webhooks/stripe', async (req, res) => {
  verifyStripeSignature(req)
  await handleStripeEventService.handle(req.body)
  res.status(204).end()
})
```

#### SSE

```ts
app.post('/support/sse', async (req, res) => {
  const stream = await supportAgentService.handle(req.body)
  res.setHeader('Content-Type', 'text/event-stream; charset=utf-8')
  res.setHeader('Cache-Control', 'no-cache')
  res.setHeader('Connection', 'keep-alive')
  res.flushHeaders?.()

  for await (const chunk of stream) {
    const data = chunk.replace(/\n/g, '\\n')
    if (!res.write(`data: ${data}\n\n`)) {
      await new Promise((resolve) => res.once('drain', resolve))
    }
  }

  res.write('event: done\ndata: [DONE]\n\n')
  res.end()
})
```

#### Message Brokers

```ts
queue.on('message', async (envelope) => {
  await redeemCardService.handle(envelope.payload)
})

// adapter can seed context from message metadata
```

#### Queues (AWS SQS)

```ts
const bus = sqsAdapter(framework, { client: sqs, queueUrl })

bus.consumer('IssueCard', 'giftCard.issue')
bus.consumer('RedeemCard', 'giftCard.redeem')

bus.start()
```

#### CLI / Background Jobs

```ts
program.command('issue-card').action(async (args) => {
  await issueCardService.handle(args)
})

scheduler.every('5 minutes', async () => {
  await reconcileLedgerService.handle({})
})
```

## Best Practices

- Adapters never execute domain handlers directly.
- Application services decide `execute` vs `publish` (temporal decoupling).
- Adapters can seed context but should not manage ALS lifecycle.
