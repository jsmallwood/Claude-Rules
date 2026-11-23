---
paths:
  - "**/*.md"
---

# Documentation Rules

<!-- Merged from documentation/documentation-rules.md and documentation/how-to-document-rules.md —
     both carried the same `paths: *.md` trigger scope; split here into the two purposes
     they actually served: project/technical docs vs. end-user how-to guides. -->

## Technical & project documentation
- Write READMEs with: quick start, install, usage examples, configuration, contributing
- Document the why in comments, not the what — see coding-style-rules.md
- Maintain an ARCHITECTURE.md or DECISIONS.md for key technical decisions (ADRs)
- Generate API reference docs from code annotations; keep them in sync with CI
- Write runbooks for all operational procedures (deploy, rollback, incident response)
- Use diagrams-as-code (Mermaid) for architecture diagrams; commit them with the code
- Document environment variables in .env.example with descriptions for all keys
- Add examples for every non-trivial function; examples are documentation too
- Write docs for non-obvious decisions and trade-offs, not just how the code works
- Update docs in the same change that alters observable behavior — see changelog-rules.md for the changelog counterpart of this rule

## End-user how-to guides
Persona: an expert technical writer creating "How To" documentation for non-technical users, converting technical information, test scripts, or screenshots into user-friendly guides.

Best practices:
1. **Clear title**: action-oriented, e.g. "How To Log In" or "How To Export Reports"
2. **Brief introduction**: the feature's purpose and value in 1-3 sentences
3. **Prerequisites**: required accounts, permissions, or prior steps
4. **Numbered steps**: a logical, step-by-step sequence
5. **Visual cues**: reference UI elements as users see them (buttons, fields, menus)
6. **Expected results**: what the user should see after each action
7. **Troubleshooting tips**: common issues and their solutions
8. **Related features**: mention related features or next steps when appropriate
9. **Platform compatibility**: note any differences between devices or platforms

Document structure: Title → Introduction → Prerequisites → Step-by-Step Instructions → Expected Results → Troubleshooting → Additional Information.

### Example
```markdown
# How To Log In to the Application

This guide explains how to log in to the application to access your account and personal dashboard.

**Prerequisites**: An active account with a verified email address.

**Steps**:
1. Open the application and click **Log In** in the top-right corner.
2. Enter your email address and password.
3. Click **Sign In**.

**Expected result**: You're redirected to your dashboard, and your name appears in the top-right menu.

**Troubleshooting**:
- *"Incorrect password" error*: Click **Forgot password?** on the login screen to reset it.
- *Nothing happens after clicking Sign In*: Check that your browser allows cookies for this site.
```
