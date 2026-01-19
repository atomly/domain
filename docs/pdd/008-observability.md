# Observability

Observability should be a first-class framework concern so workflows remain understandable across time. The framework should own trace propagation, span boundaries, and signal exports so application code can focus on domain logic.

## Responsibilities

- Extract and propagate W3C trace context across HTTP, queues, and event handlers.
- Create spans around application services, command execution, event handling, and query handling.
- Correlate commands/events/queries with trace identifiers to preserve causality.
- Export traces, metrics, and logs via OpenTelemetry/OTLP or equivalent adapters.

## Standards

- **W3C Trace Context** (`traceparent`, `tracestate`)
- **OpenTelemetry** instrumentation APIs and semantic conventions
- **OTLP** as the primary export protocol for traces, metrics, and logs

## Adapters

Observability adapters translate framework signals into concrete tooling while keeping the surface area small.

### Logging

Logging adapters emit structured logs with request-scoped context (trace ID, boundary, command/event name). They should support configurable levels and pluggable formatters (JSON, logfmt).

```ts
export abstract class LoggerAdapter {
  abstract info(message: string, fields?: Record<string, unknown>): void
  abstract warn(message: string, fields?: Record<string, unknown>): void
  abstract error(message: string, fields?: Record<string, unknown>): void
}
```

### Metrics

Metrics adapters support counters, gauges, and histograms with tagging. The framework can record handler durations, error counts, and queue lag automatically while still allowing custom metrics.

```ts
export abstract class MetricsAdapter {
  abstract counter(name: string, value: number, tags?: Record<string, string>): void
  abstract gauge(name: string, value: number, tags?: Record<string, string>): void
  abstract histogram(name: string, value: number, tags?: Record<string, string>): void
}
```

### Tracing

Tracing adapters centralize span creation and context propagation. The framework can auto-wrap application services, handlers, and query paths while still allowing custom spans.

```ts
export abstract class TracingAdapter {
  abstract withSpan<T>(name: string, fn: () => Promise<T>): Promise<T>
}
```

## Integration Notes

- Framework and router instrumentation should share one OpenTelemetry SDK so spans land in the same pipeline.
- Adapter registration happens at boot; request-scoped access uses `useLogger`, `useMetrics`, and `useTracer` via the provider registry.
- Application code should not manually construct tracing metadata.
