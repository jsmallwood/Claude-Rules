# Observability with OpenTelemetry Rules

<!-- Canonical source for metric types/labeling and tracing conventions.
     error-handling-rules.md's "emit a metric per error type" cross-references this file. -->

## The Three Pillars
- **Traces**: distributed request flow across services — OpenTelemetry SDK
- **Metrics**: aggregated measurements — Prometheus + Grafana or Datadog
- **Logs**: structured events — always JSON, always with trace ID

## Instrumentation Setup
```typescript
// Initialize BEFORE importing application code
import { NodeSDK } from '@opentelemetry/sdk-node';
import { OTLPTraceExporter } from '@opentelemetry/exporter-trace-otlp-http';

const sdk = new NodeSDK({
  traceExporter: new OTLPTraceExporter({ url: process.env.OTEL_ENDPOINT }),
  instrumentations: [getNodeAutoInstrumentations()], // HTTP, express, DB auto-instrumented
});
sdk.start();
```

## Trace Conventions
- Service name: lowercase, hyphenated (`user-service`, `payment-api`)
- Every HTTP request: automatic via auto-instrumentation
- Custom spans for business operations: `tracer.startActiveSpan('process-payment', ...)`
- Span attributes: `user.id`, `order.id`, `payment.amount` — queryable in Jaeger/Tempo
- Error spans: `span.setStatus({ code: SpanStatusCode.ERROR, message: err.message })`

## Structured Logging
```typescript
// Every log includes trace context
const logger = {
  info: (msg: string, attrs?: object) => {
    const span = trace.getActiveSpan();
    const ctx = span?.spanContext();
    console.log(JSON.stringify({
      level: 'info', message: msg, timestamp: new Date().toISOString(),
      traceId: ctx?.traceId, spanId: ctx?.spanId,
      ...attrs,
    }));
  },
};
```

## Metrics
- Use counters for events (`requests_total`), histograms for durations (`request_duration_ms`), gauges for point-in-time values (`active_connections`)
- Attach low-cardinality labels only (`route`, `status_code`) — never put user IDs or free-text in a metric label, it blows up cardinality
- Export via OTLP to a collector rather than each service pushing directly to the backend, so exporters/backends can change without redeploying services

## Sampling
- Use head-based sampling (e.g. 10% of traces) in high-volume production services; always sample 100% of error traces regardless of the base rate
- Propagate the sampling decision via trace context headers (`traceparent`) so a single request is sampled consistently across services
- Never sample locally per-service — inconsistent sampling breaks distributed trace reconstruction
