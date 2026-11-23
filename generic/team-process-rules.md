# Startup Team (2-8 Devs) Process Rules

<!-- Renamed from generic/startup-team-rules.md. "Communication in Code" trimmed —
     the comment/commit-message convention now lives in coding-style-rules.md. -->

## Code Ownership & Reviews
- Solo project: self-review your own diff before merging to main
- PR size limit: 400 lines of production code (tests separate)

## Branching Strategy
- Trunk-based development with short-lived feature branches (<3 days)
- Feature flags for incomplete features merged to main
- No long-lived feature branches — they cause painful merges
- Branch naming: `{type}/{short-description}`

## Definition of Done
- [ ] Feature works as specified
- [ ] Tests written (unit + integration for business logic)
- [ ] Diff self-reviewed
- [ ] No TypeScript/lint errors
- [ ] Environment variables documented in .env.example
- [ ] Feature flag added if partially complete

## Communication in Code
- Commit messages and comments explain WHY, not WHAT — see coding-style-rules.md for the general comment convention
- Link to issue in commit or PR body when one exists

## Shared Ownership Areas
- Auth system: whole team understands it — no single expert
- Database schema: migrations reviewed by at least 2 people
- External integrations (payments, email): runbook exists before going live
- Deployment process: any 2 team members can deploy independently

## Incident Response
- Runbook for the 5 most likely outages — written before they happen
- On-call rotation documented and communicated
- Post-mortem for any outage >30 minutes — blameless, action-item focused
