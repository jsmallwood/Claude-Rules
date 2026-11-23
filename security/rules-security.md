# Security Guidelines

<!-- Canonical general application-security baseline (merged from the former security-rules.md).
     Domain-specific files (github-actions-rules.md, graphql-design-rules.md, native-mobile-rules.md,
     sql-rules.md) cross-reference this file for shared concerns (secrets, rate limiting,
     input validation) and keep only what's genuinely specific to their domain. -->

## Mandatory Security Checks

Before ANY commit:
- [ ] No hardcoded secrets (API keys, passwords, tokens)
- [ ] All user inputs validated
- [ ] SQL injection prevention (parameterized queries)
- [ ] XSS prevention (sanitized HTML)
- [ ] CSRF protection enabled
- [ ] Authentication/authorization verified
- [ ] Rate limiting on all endpoints
- [ ] Error messages don't leak sensitive data

## Baseline Practices

- Treat all external input as untrusted; validate shape, type, and range
- Use environment-specific secret management (Vault, AWS Secrets Manager, Doppler)
- Implement proper session management: short-lived tokens, refresh rotation
- Use constant-time comparison for security-sensitive string equality checks
- Apply Content Security Policy and review it regularly
- Audit dependencies weekly with npm audit, pip-audit, or cargo audit
- Log security events (failed logins, permission errors) to an immutable audit log
- Perform threat modeling before building new authentication flows
- Hash passwords with bcrypt/argon2 (never MD5/SHA); use a per-user salt and a work factor tuned to ~200-500ms per hash
- Validate JWTs on every request: verify signature, expiry, issuer, and audience; never trust unverified claims
- Generate CSRF tokens for state-changing requests on any endpoint that accepts cookie-based auth
- Apply rate limiting per-user and per-IP on auth endpoints and any expensive operation, not just globally

## Secret Management

```typescript
// NEVER: Hardcoded secrets
const apiKey = "sk-proj-xxxxx"

// ALWAYS: Environment variables
const apiKey = process.env.OPENAI_API_KEY

if (!apiKey) {
  throw new Error('OPENAI_API_KEY not configured')
}
```

## Security Response Protocol

If security issue found:
1. STOP immediately
2. Use **security-reviewer** agent
3. Fix CRITICAL issues before continuing
4. Rotate any exposed secrets
5. Review entire codebase for similar issues