---
paths:
  - "**/services/**"
  - "**/Dockerfile*"
  - "**/docker-compose*.{yml,yaml}"
---

# Microservices Architecture Rules

## Service Boundaries
- Services own their data — no shared databases between services
- Each service has one clear responsibility — if you need "and," split it
- Service size: 1-2 developers can understand the full codebase in a day
- Domain-driven design boundaries align with team ownership
- Avoid distributed monolith: if services can't deploy independently, you have a monolith

## Inter-Service Communication
- Synchronous (HTTP/gRPC): for reads that need immediate response
- Asynchronous (message queue): for writes, events, notifications
- Never call >2 services in a synchronous chain — latency compounds
- Circuit breaker pattern for all downstream service calls
- Timeout on every external call — never wait indefinitely

```typescript
// Circuit breaker with cockatiel or opossum
const breaker = new CircuitBreaker(callUserService, {
  timeout: 3000,           // Fail after 3s
  errorThresholdPercentage: 50,  // Open at 50% failure rate
  resetTimeout: 30000,     // Try again after 30s
});
```

## Event-Driven Communication
- Events are facts: named in past tense (`OrderPlaced`, `UserCreated`)
- Event schema versioning — consumers must handle old and new versions
- Event store for audit trail — every state change as an event
- Dead letter queue (DLQ) for failed event processing — never silently drop
- At-least-once delivery with idempotent consumers

## API Gateway
- Single entry point for all client traffic
- Authentication and rate limiting at the gateway — not per service
- Request routing, load balancing, SSL termination
- Request/response transformation and protocol translation
- Circuit breaking and retry logic at the gateway level