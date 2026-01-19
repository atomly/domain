# Controllers

Controllers connect the domain to the outside world. They adapt transports (HTTP, WebSockets, queues, CLI) to application services so user interfaces and external consumers can interact with the domain safely and consistently. API styles like REST and GraphQL ride on top of these transports.

## Layer Responsibilities

Separating responsibilities keeps transport details out of domain logic:

- **Controllers**: parse inputs, manage headers/status codes, handle streaming, map errors, and inject request context.
- **Application Services**: orchestrate use cases and call handlers/adapters.
- **Command Handlers**: mutate state and emit events.
- **Query Handlers**: read models and shape responses.

## Transports

Controllers are transport adapters. Each one maps a transport’s request/response shape to application service inputs and outputs, reusing the same use cases across protocols. Here are some examples for each transport type:

### REST

```ts
app.get('/users/:id', async (req, res) => {
	const result = await getUserService.handle({ id: req.params.id })
	res.json(result)
})
```

### GraphQL

```ts
const resolvers = {
	Query: {
		user: (_: unknown, args: { id: string }) => getUserService.handle(args)
	},
	Mutation: {
		createUser: (_: unknown, args: CreateUserInput) =>
			createUserService.handle(args)
	}
}
```

### RPC (JSON-RPC / gRPC)

```ts
rpcServer.on('request', async (req) => {
	if (req.method === 'user.get') return getUserService.handle(req.params)
	if (req.method === 'user.create') return createUserService.handle(req.params)
	throw new RpcError('METHOD_NOT_FOUND')
})
```

### WebSockets (Real-time Gateways)

```ts
wsServer.on('message', async (message) => {
	const input = JSON.parse(message.toString())
	const result = await issueCardService.handle(input)
	wsServer.send(JSON.stringify(result))
})
```

### Webhooks (Inbound Integrations)

```ts
app.post('/webhooks/stripe', async (req, res) => {
	verifyStripeSignature(req)
	await handleStripeEventService.handle(req.body)
	res.status(204).end()
})
```

### SSE

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

### Message Brokers

```ts
queue.on('message', async (envelope) => {
	await redeemCardHandler.execute(envelope.payload)
})
```

### CLI / Background Jobs

```ts
program.command('issue-card').action(async (args) => {
	await issueCardService.handle(args)
})

scheduler.every('5 minutes', async () => {
	await reconcileLedgerService.handle({})
})
```

## HTTP Controllers (Express/Fastify)

HTTP controllers map verbs to application services and treat services as pure execution boundaries. REST routes are one subset of HTTP, while streaming endpoints (SSE/WebSockets) share the same server runtime.

### End-to-end Example (REST)

```ts
// controller
app.post('/users', async (req, res) => {
	const result = await createUserService.handle(req.body)
	res.status(201).json(result)
})

// application service
export const createUserService = defineApplicationService()
	.input(CreateUserInput)
	.handle(async (input) => {
		await createUserHandler.execute(input)
		return { ok: true }
	})

// command handler
export const createUserHandler = defineCommandHandler()
	.entity(User)
	.command(CreateUser)
	.creation('always')
	.handle((command, state) => {
		state.id = command.id
		state.email = command.email
		raise(UserCreated, { id: command.id })
	})
```

### Express (REST CRUD)

```ts
app.get('/users/:id', async (req, res) => {
	const result = await getUserService.handle({ id: req.params.id })
	res.json(result)
})

app.get('/users', async (req, res) => {
	const result = await listUsersService.handle({
		limit: req.query.limit,
		cursor: req.query.cursor
	})
	res.json(result)
})

app.post('/users', async (req, res) => {
	const result = await createUserService.handle(req.body)
	res.status(201).json(result)
})

app.put('/users/:id', async (req, res) => {
	const result = await updateUserService.handle({
		id: req.params.id,
		...req.body
	})
	res.json(result)
})

app.patch('/users/:id', async (req, res) => {
	const result = await patchUserService.handle({
		id: req.params.id,
		...req.body
	})
	res.json(result)
})

app.delete('/users/:id', async (req, res) => {
	await deleteUserService.handle({ id: req.params.id })
	res.status(204).end()
})
```

### Fastify (REST CRUD)

```ts
fastify.get('/users/:id', async (request, reply) => {
	const result = await getUserService.handle({ id: request.params.id })
	return reply.send(result)
})

fastify.get('/users', async (request, reply) => {
	const result = await listUsersService.handle({
		limit: request.query.limit,
		cursor: request.query.cursor
	})
	return reply.send(result)
})

fastify.post('/users', async (request, reply) => {
	const result = await createUserService.handle(request.body)
	return reply.code(201).send(result)
})

fastify.put('/users/:id', async (request, reply) => {
	const result = await updateUserService.handle({
		id: request.params.id,
		...request.body
	})
	return reply.send(result)
})

fastify.patch('/users/:id', async (request, reply) => {
	const result = await patchUserService.handle({
		id: request.params.id,
		...request.body
	})
	return reply.send(result)
})

fastify.delete('/users/:id', async (request, reply) => {
	await deleteUserService.handle({ id: request.params.id })
	return reply.code(204).send()
})
```

### Streaming (SSE + WebSockets)

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

wsServer.on('message', async (message) => {
	const input = JSON.parse(message.toString())
	const result = await supportAgentService.handle(input)
	wsServer.send(JSON.stringify(result))
})
```

## Framework Integrations

Rather than owning the full controller stack, we should integrate directly with established routers. For v0, the preferred path is oRPC because it gives us adapters, OpenAPI support, and streaming without additional framework complexity. A small `@framework/orpc-controller` package can provide the glue (base procedure, context wiring, and observability hooks) so teams don’t need to hand-roll this integration. Observability standards and adapter responsibilities live in [`docs/pdd/007-observability.md`](007-observability.md).

### oRPC Integration

The controller package would expose helpers that connect oRPC’s context to the framework and attach tracing + error handling to oRPC handlers.

```ts
// @framework/orpc-controller (conceptual)
export const base = (framework: Framework) =>
	os.$context<{ headers: IncomingHttpHeaders }>().use(({ context, next }) =>
		next({ context: { ...context, framework: framework.context() } })
	)

export const withTracing = os.middleware(async ({ context, next }) => {
	const span = trace.getActiveSpan()
	span?.setAttribute('domain.framework', 'domain-first')
	return next({ context })
})

export const createRpcHandler = (router: unknown, logger: LoggerAdapter) =>
	new RPCHandler(router, {
		interceptors: [
			onError((error) => logger.error(error)),
			({ request, next }) => {
				const span = trace.getActiveSpan()
				request.signal?.addEventListener('abort', () => {
					span?.addEvent('aborted', { reason: String(request.signal?.reason) })
				})
				return next()
			}
		]
	})

export const createOpenApiHandler = (router: unknown) =>
	new OpenAPIHandler(router)
```

We would boot OpenTelemetry once and register adapters so oRPC spans and framework spans share the same SDK and exporters.

```ts
// framework.ts
new NodeSDK({
	instrumentations: [new ORPCInstrumentation(), new FrameworkInstrumentation()]
}).start()

export const framework = createFramework()
	.provide(LoggerAdapter, new PinoLoggerAdapter())
	.provide(MetricsAdapter, new OtelMetricsAdapter())
	.provide(TracingAdapter, new OpenTelemetryAdapter())
	.provide(CommandPublisher, new LocalCommandPublisher())
```

Procedures would reuse the shared base context and optional tracing middleware, then call application services.

```ts
// router.ts
const listUsers = base(framework)
	.use(withTracing)
	.input(ListUsersInput)
	.handler(({ input }) => listUsersService.handle(input))

const router = { listUsers }
```

We would mount the RPC handler, forward headers into the oRPC context, and fall through when the route does not match.

```ts
// app.ts
const handler = createRpcHandler(framework)(router)

app.use('/rpc/*', async (req, res, next) => {
	const { matched } = await handler.handle(req, res, {
		prefix: '/rpc',
		context: { headers: req.headers }
	})

	if (matched) return
	next()
})
```
