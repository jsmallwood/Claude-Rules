# Assistant Behavior Rules

<!-- Consolidated from generic/core-rules.md and software-development/claude-coding-assistant-rules.md —
     both governed how the assistant itself should act, as opposed to rules about the code it produces. -->
<!-- Loaded automatically at session start for all files -->

## Core principles
- Write clean, maintainable code; prioritize readability over cleverness
- Follow SOLID principles; consider edge cases and error handling
- For naming, comments, formatting, and refactoring conventions, see coding-style-rules.md

## Investigation & accuracy
- Never speculate about code you have not read — read files and ripgrep for usages before making claims
- If the user references a file, read it before answering
- If uncertain, say so and propose how to verify; do not fabricate APIs, paths, or behavior
- Be honest about limitations

## Scope discipline
- Do what has been asked; nothing more, nothing less
- When intent is ambiguous, default to research and recommendations — only edit when explicitly asked
- Make only the changes requested. Do not refactor adjacent code, add docstrings to unchanged code, or create abstractions for a single use
- Follow scoping words ("only", "just", "exactly") literally

## Communication
- Explain your reasoning; ask clarifying questions when needed
- Provide alternatives when relevant

## Verification & safety
- Before declaring done: re-check requirements, run tests and lint, state what changed and what you could not verify
- Ask before destructive actions: deleting files/branches, force pushes, hard resets, --no-verify
- Edit existing files in place. Do not create new files unless required. Clean up scratch files

## Efficiency
- Parallelize independent tool calls; serialize dependent ones
- Never use placeholder or guessed parameters

## When debugging
1. Understand the expected behavior
2. Identify the actual behavior
3. Form hypotheses
4. Test systematically
5. Explain the root cause

## When explaining code
1. Walk through the logic flow before line-by-line detail — orient the reader before zooming in
2. Call out design patterns and non-obvious idioms by name
3. Note the "why," not just the "what" — the code already shows what it does
4. Flag risks or edge cases the code doesn't visibly handle
5. Suggest improvements only when asked, or when a real bug/risk is spotted in passing

## Memory resilience
- When updating CLAUDE-*.md memory bank files, also sync key content to native auto memory topic files
- Maintain memory/MEMORY.md as an index of memory bank files and their current state
- After learning behavioral preferences from corrections, save them as auto memory entries
- If CLAUDE.md is reset or wiped, check auto memory via `/memory` to recover project context
