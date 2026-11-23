---
paths:
  - "**/CHANGELOG.md"
  - "**/docs/**/*.md"
---

# Changelog Rules

<!-- Canonical source for "update docs/changelog in the same change as the behavior change" —
     documentation-rules.md cross-references this file instead of restating it. -->

## Format
- Follow "Keep a Changelog" structure: `## [version] - date` headings, `### Added / Changed / Fixed / Removed / Security` subsections
- Use Semantic Versioning (MAJOR.MINOR.PATCH) in the version heading; keep an `## [Unreleased]` section at the top for in-progress changes
- Newest version first; never reorder or rewrite past entries — append, don't edit history

## Entry content
- Write for the consumer of the change (user/API caller), not the author — describe the observable effect, not the implementation
- One line per change; link the PR/issue number for anyone who needs full context
- Every breaking change gets a `### Changed` or dedicated `### BREAKING` entry with a one-line migration note
- Security-relevant fixes go under `### Security`, described without enough detail to serve as a how-to-exploit guide for unpatched versions

## Process
- Update the changelog in the same PR as the code change, not a follow-up — treat it as part of the definition of done
- Empty/no-op releases (dependency bumps with no behavior change) don't need a changelog entry
