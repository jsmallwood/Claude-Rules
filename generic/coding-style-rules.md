# Coding Style Rules

<!-- Canonical home for naming/comment/formatting conventions. Other rule files
     (assistant-behavior, documentation, team-process) cross-reference this file
     instead of restating "comment the why, not the what".
     Merged with the former root rules-coding-style.md (immutability, size limits, checklist).
     Error handling lives in error-handling-rules.md; input validation in rules-security.md. -->

## Naming
- Use each language's native convention (snake_case for Python, camelCase for JS/TS variables, PascalCase for classes/components) — see the per-language rule files for specifics; don't override them here
- Name things for what they represent in the domain, not their type or implementation (`activeUsers`, not `userList` or `userArr`)
- Boolean names read as a question/predicate: `isActive`, `hasPermission`, `canEdit`

## Formatting & linting
- Enforce style with tooling (Prettier/Biome/Ruff/Black), not review comments — if a formatting nit shows up in review, add a lint rule instead of repeating the comment
- Run lint/format as a pre-commit hook and in CI so violations never reach `main`
- Never hand-format around an auto-formatter's output; if the formatter's output looks wrong, fix the config, not the instance

## Immutability
- Create new objects instead of mutating inputs: return `{ ...user, name }`, not `user.name = name; return user`
- Mutate only locally scoped data (e.g. a list being built inside one function)

## File & module size
- Typical file 200-400 lines, 800 max; organize by feature/domain, not by type
- Functions stay small (<50 lines) with no deep nesting (>4 levels)
- Keep files focused on one responsibility; a file needing a "utils" or "helpers" catch-all section is a signal to split it
- Prefer several small, well-named modules over one large mixed-responsibility file
- Extract a function/module when it's reused twice, not preemptively for a single use

## Comments
- Comment the "why," not the "what" — code already shows what it does; a comment should explain a non-obvious constraint, workaround, or trade-off
- Delete commented-out code instead of leaving it "just in case" — version control already has the history
- Keep comments next to the code they describe and update them in the same change that changes the code

## Imports
- Group and order imports: standard library → third-party → internal/local, with a blank line between groups
- Use absolute imports for cross-module references; relative imports only within a tightly-coupled local package
- Avoid wildcard imports (`from x import *`) except in narrowly-scoped, well-understood cases (e.g. re-export barrels)

## Refactoring checklist
- Extract functions when a block does more than one thing or is reused
- Remove duplication by extracting the shared logic, not by copy-pasting with small edits
- Simplify nested conditionals with early returns/guard clauses
- Improve names before restructuring — often clarifies whether restructuring is even needed
- Add/tighten type annotations as part of the refactor, not as a separate pass
- Only introduce a new abstraction when the duplication it removes is real, not speculative
- Update or add tests alongside the refactor so behavior preservation is verified, not assumed
- Note non-obvious refactors (behavior-preserving but structurally large) in the PR description

## Before marking work complete
- [ ] Code is readable and well-named
- [ ] Functions small, files focused, nesting shallow (limits above)
- [ ] Errors handled per error-handling-rules.md; user input validated per rules-security.md
- [ ] No leftover debug output (`console.log`, `print`) — use the project's logger
- [ ] No hardcoded values that belong in config or env vars
- [ ] No unintended mutation
