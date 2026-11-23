# Error Handling Rules

<!-- Canonical home for the retry/backoff/circuit-breaker policy. performance-rules.md and
     native-mobile-rules.md cross-reference this file instead of restating it. -->

## Error hierarchies
- Define a small set of typed/custom error classes (e.g. `ValidationError`, `NotFoundError`, `UnauthorizedError`) rather than throwing generic `Error`/`Exception`
- Attach a stable error code to each type so callers/clients can branch on it without string-matching messages

## Catch vs. propagate
- Only catch an error where you can meaningfully handle it (retry, fall back, translate to a user-facing message) — otherwise let it propagate
- Never swallow an error silently (empty catch block); at minimum log it before re-throwing or returning a failure result
- Catch at service/handler boundaries to translate internal errors into the external contract (HTTP status + body, RPC error, etc.), not deep inside business logic

## User-facing messages & logging
- Return actionable, specific error messages to callers; never leak stack traces, internal paths, or secrets in responses
- Log full context (stack trace, request ID, relevant input) server-side even when the client-facing message is generic
- Correlate errors to requests with a request/trace ID so a reported error can be found in logs

## Recovery & retries
- Retry only idempotent operations, with exponential backoff and jitter, and a max-attempt cap
- Distinguish transient errors (network timeout, 503) from permanent ones (validation failure, 404) — only retry the former
- Provide a fallback/degraded path for non-critical dependencies rather than failing the whole request
- This is the canonical retry policy — apply it wherever a domain rule file calls for "retry with backoff" (e.g. performance-rules.md, native-mobile-rules.md)

## Observability
- Emit a metric/counter per error type so spikes are visible on a dashboard, not just in logs — see observability-rules.md for metric types and labeling conventions
- Alert on error-rate thresholds, not individual errors, to avoid alert fatigue
