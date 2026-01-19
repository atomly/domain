# AI Integrations

This document captures how the framework should support AI agents and tools without providing its own AI library. The framework should treat AI providers as secondary adapters and keep domain boundaries explicit.

## Goals

- Support AI SDKs like Vercel's AI SDK without re-implementing them.
- Keep AI usage in application services or domain services, not entities or handlers.
- Validate AI outputs before they become commands or events.
- Support streaming responses end-to-end (AsyncIterable/ReadableStream).
- Emit observability signals for tokens, latency, and cost.

## Adapter Shape

AI providers should be wired as adapters so the framework can stay vendor-neutral while still enabling direct use of external SDKs.

```ts
import {
	generateText,
	generateObject,
	streamText,
	type LanguageModel
} from 'ai'

export abstract class AiProvider {
	abstract generateText(input: Parameters<typeof generateText>[0]): Promise<string>
	abstract generateObject<T>(input: Parameters<typeof generateObject>[0]): Promise<T>
	abstract streamText(
		input: Parameters<typeof streamText>[0]
	): Promise<AsyncIterable<string>>
}

export class VercelAiProvider extends AiProvider {
	constructor(private model: LanguageModel) {
		super()
	}

	async generateText(input: Parameters<typeof generateText>[0]) {
		const result = await generateText({ ...input, model: this.model })
		return result.text
	}

	async generateObject<T>(input: Parameters<typeof generateObject>[0]) {
		const result = await generateObject({ ...input, model: this.model })
		return result.object as T
	}

	async streamText(input: Parameters<typeof streamText>[0]) {
		const result = await streamText({ ...input, model: this.model })
		return result.textStream
	}
}
```

## Tool Integration

Tools should map to domain queries/commands and live outside the AI SDK. The provider can accept a tool list from a registry.

```ts
import { tool, type Tool } from 'ai'

export abstract class ToolRegistry {
	abstract list(): Tool[]
}

export class InMemoryToolRegistry extends ToolRegistry {
	constructor(private tools: Tool[]) {
		super()
	}

	list() {
		return this.tools
	}
}
```

## Streaming Application Services

Application services should be allowed to return streams so transports can stream responses instead of buffering the full output.

```ts
export const supportAgentService = defineApplicationService()
	.input(SupportQueryInput)
	.handle(async (input) => {
		const ai = useAdapter(AiProvider)
		return ai.streamText({
			model: openai('gpt-4o-mini'),
			prompt: input.message
		})
	})
```

## Observability

AI integrations should emit consistent signals (span names, attributes, metrics, logs) through the framework observability adapters described in [`docs/pdd/007-observability.md`](007-observability.md).

