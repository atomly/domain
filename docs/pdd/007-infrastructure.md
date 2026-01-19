# Infrastructure

This document sketches how adapters and providers could be registered and resolved using ALS-powered context, while keeping domain code ergonomic and explicit.

## Adapters

Adapters should connect the framework to infrastructure concerns (databases, messaging, observability, etc.). For v0 we should use **explicit registration** instead of auto-registration to keep behavior transparent and debuggable. Abstract classes should act as ports, while concrete classes implement the adapters. Providers are the DI layer that supplies these adapters (and other app services) at runtime.

- Provide a DI-like experience without hard dependencies on a DI framework.
- Use providers as the DI mechanism, while adapters remain the port/adapter abstraction.
- Keep domain code functional while allowing adapters to be class-based.
- Resolve adapters from request scope via ALS.

### Dependency Injection

Use abstract classes as DI tokens. Implementations should be registered with the framework and resolved via core APIs such as `useRepository`, `useLogger`, `useMetrics`, `useTracer`, or `useProvider` for custom providers.

Register implementations at the framework boundary:

```ts
const framework = createFramework()
	.provide(GiftCardRepository, new PostgresGiftCardRepository(pg))
	.provide(LoggerAdapter, new PinoLoggerAdapter())
	.provide(MetricsAdapter, new OtelMetricsAdapter(otel))
	.provide(TracingAdapter, new OpenTelemetryAdapter(otel))
	.provide(CommandPublisher, new LocalCommandPublisher())
```

This fluent builder style mirrors modern DX patterns in tools like tRPC and TanStack by keeping configuration declarative and discoverable. Provider registration can be backed by an Effect-style `provideService` mechanism.

Resolve inside handlers and services:

```ts
const giftCardRepository = useRepository(GiftCardRepository)
const logger = useLogger()
const metrics = useMetrics()
const tracer = useTracer()
```

This keeps domain code clean while relying on explicit adapter wiring at boot.

Alternate possible explicit registration API (if the builder pattern is not available):

```ts
const registry = createProviderRegistry()

registry.registerRepository(GiftCardRepository, new PostgresGiftCardRepository(pg))
registry.registerLogger(new PinoLoggerAdapter())
registry.registerMetrics(new OtelMetricsAdapter(otel))
registry.registerTracing(new OpenTelemetryAdapter(otel))
registry.registerCommandPublisher(new LocalCommandPublisher())

const framework = createFramework({ providers: registry })
```

### ALS-backed APIs

Context should be request-scoped and resolved via ALS. Hook-style helpers hide the context plumbing but remain explicit about dependencies. The ALS context can power their implementations, but the API should stay small.

Utilities could include:

- `raise`
- `withAuth` or `requireUser`

Hooks should make the dependency lookup explicit while signaling ALS-backed resolution. The `use` prefix mirrors common hook conventions to keep the implicit context obvious without pushing `ctx` through every call; alternatives like `resolveRepository` or `getLogger` could work, but `use*` reads clearly as a request-scoped lookup.

Hooks should include:

- `useRepository`
- `useLogger`
- `useMetrics`
- `useTracer`
- `useProvider`

Example shorthand:

```ts
raise(CardIssued, payload)
```

### Provider Overrides

Request-scoped provider overrides can be defined at the application-service boundary without pushing wiring into handlers. Providers can be backed by an Effect `provideService`-style registry to keep DI explicit but lightweight.

```ts
export const issueCardService = defineApplicationService()
  .input(
    z.object({
      amount: z.number().int().positive()
    })
  )
  .provide(GiftCardIssuer, new InMemoryGiftCardIssuer())
  .handle(async (input) => {
    const cardId = crypto.randomUUID()

    await issueCardHandler.execute({ cardId, amount: input.amount })

    return { cardId }
  })
  ```

### Core Framework Adapters

Core adapters should be few and explicit. They adapt the framework’s internal abstractions (commands, events, persistence, observability) to the infrastructure where those concerns actually live. In other words, adapters translate domain-friendly concepts into concrete transports, protocols, and data stores so the runtime can remain portable across environments.

The list below is not exhaustive, but it captures the key integration points and naming. The examples below illustrate the general DI pattern.

- `CommandPublisher`: publishes command envelopes to the configured transport.
- `CommandSubscriber`: subscribes to command transports and hands envelopes to execution.
- `EventPublisher`: forwards raised domain events to the outbound transport.
- `EventSubscriber`: consumes inbound events and invokes event handlers.
- `LoggerAdapter`: structured logging with correlation context.
- `MetricsAdapter`: counters, gauges, histograms, and summaries.
- `TracingAdapter`: spans, context propagation, and OTEL export. (See [`docs/pdd/008-observability.md`](008-observability.md).)
- `EntityRepository`: persistence port for loading/saving entity state.

#### Repository Adapters

Repository adapter details live in [`docs/pdd/006-repositories.md`](006-repositories.md). Infrastructure still registers repositories through the provider registry so they can be resolved via `useRepository`.

#### Transport Adapters

Transport adapters bridge the framework’s command/event pipeline with infrastructure. Outbound adapters publish command/event envelopes, while inbound adapters receive envelopes and invoke the appropriate handler. This keeps the handler API stable while allowing different transports (in-process, queues, streams, serverless triggers) to be configured at boot.

- Outbound: `CommandPublisher`, `EventPublisher`
- Inbound: `CommandSubscriber`, `EventSubscriber`

Example (in-process transport):

```ts
import { EventEmitter } from 'node:events'

export class LocalCommandPublisher extends CommandPublisher {
	private emitter = new EventEmitter()

	async publish(envelope: CommandEnvelope) {
		this.emitter.emit('command', envelope)
	}
}

export class LocalCommandSubscriber extends CommandSubscriber {
	constructor(private emitter: EventEmitter) {
		super()
	}

	bind() {
		this.emitter.on('command', (envelope) => commandHandler.execute(envelope))
	}
}

export class LocalEventPublisher extends EventPublisher {
	private emitter = new EventEmitter()

	async publish(envelope: EventEnvelope) {
		this.emitter.emit('event', envelope)
	}
}

export class LocalEventSubscriber extends EventSubscriber {
	constructor(private emitter: EventEmitter) {
		super()
	}

	bind() {
		this.emitter.on('event', (envelope) => eventHandler.handle(envelope))
	}
}
```

#### Observability Adapters

Observability adapter details live in [`docs/pdd/008-observability.md`](008-observability.md). Infrastructure still registers the adapters through `createFramework().provide(...)` so logging, metrics, and tracing stay consistent across transports.

### Secondary Adapters

Secondary adapters should stay boundary-owned and domain-named, typically wrapping third-party APIs or internal services. They should be wired through the same DI registry but are not part of the core framework surface.

- `PasswordHasher`
- `FraudScoreProvider`
- `PaymentProcessor`

Example wiring at boot:

```ts
import argon2 from 'argon2'

export abstract class PasswordHasher {
	abstract hash(password: string): Promise<string>
	abstract verify(hash: string, password: string): Promise<boolean>
}

export class Argon2PasswordHasher extends PasswordHasher {
	async hash(password: string) {
		return argon2.hash(password)
	}

	async verify(hash: string, password: string) {
		return argon2.verify(hash, password)
	}
}

const framework = createFramework()
	.provide(PasswordHasher, new Argon2PasswordHasher())
```

## Notes

- Tokens can be abstract classes or unique symbols.
- The framework runtime owns adapter registration and context lifetime.
- ALS enables this without threading `ctx` through every handler.
