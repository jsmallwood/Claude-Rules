# Performance Rules

<!-- Retry/timeout/circuit-breaker specifics now live in error-handling-rules.md;
     this file keeps only what's specific to performance work. -->

- Always benchmark before and after optimization to confirm improvement
- Use flamegraphs to identify CPU hotspots; trace to identify I/O bottlenecks
- Cache at the nearest layer to the user: browser → CDN → app → database
- Use read replicas for read-heavy workloads; shard for write-heavy workloads
- Implement request coalescing to prevent cache stampede under load
- Use streaming for large responses; avoid loading full datasets into memory
- Set aggressive timeouts on all external calls; implement circuit breakers — pair with the retry/backoff policy in error-handling-rules.md
- Load test with k6 or Locust before shipping major features

## Frontend
- Lazy-load routes/components and below-the-fold images; don't ship JS for UI the user hasn't reached yet
- Memoize expensive renders/computations, but measure first — unnecessary memoization adds complexity without payoff
- Serve static assets through a CDN with long cache lifetimes plus content-hashed filenames for cache busting
