---
paths:
  - "**/*.ps1"
  - "**/*.psm1"
  - "**/*.psd1"
---

# PowerShell Naming Convention Rules

These rules apply to every PowerShell file, whatever version it targets. Class-member naming details are in
`powershell-classes-rules.md`, script file naming in `powershell-scripts-rules.md`, and test naming in
`powershell-testing-rules.md`.

## Conventions

| Element | Convention | Examples |
| --- | --- | --- |
| Variables (local) | camelCase | `$pageNumber`, `$apiHeader`, `$responseBody` |
| Parameters (functions and scripts) | PascalCase | `param([string]$IssueTitle)`, called as `-IssueTitle` |
| Classes | PascalCase | `class GitHubIssue`, `class IssueFilter` |
| Enum types | PascalCase | `enum IssueState` |
| Enum values | camelCase | `open`, `closed`, `inProgress` |
| Constants | camelCase | `$maxPageSize`, `$defaultApiVersion` |
| Public functions | Approved verb + `-` + PascalCase noun (module prefix) | `Get-GtIssue`, `New-GtIssue`, `Set-GtIssue`, `Remove-GtIssue`, `Invoke-GtIssueAction` |
| Private functions | Same verb-noun convention, **not exported** | `Get-GtApiHeader`, `ConvertTo-GtIssueObject` |
| Module folders / manifest | PascalCase, folder name = module name | `GitHubTools/GitHubTools.psd1` |
| Function / class files | Named exactly after the function or type | `Public/Get-GtIssue.ps1`, `Classes/GitHubIssue.ps1` |
| Test files | `<Name>.Tests.ps1` | `tests/Public/Get-GtIssue.Tests.ps1` |
| Configuration keys | `<Module>.<Area>.<Setting>`, PascalCase segments | `GitHubTools.Api.BaseUri` |
| Error IDs | PascalCase, module prefix | `GtIssueNotFound` |
| Log tags | lowercase | `'api'`, `'retry'` |

- Prefix each noun with the module's short prefix (for example, `Gt` for a GitHub tools module) to avoid name collisions.
- Use singular nouns (`Get-GtIssue`, not `Get-GtIssues`).
- Don't use Hungarian notation or type prefixes (`$strName`, `$arrItems`).
- **Never give a local variable the same name as a parameter, in any letter case.** Variable names are case-insensitive, so `$body` and `$Body` are the same variable: a body-level `$settings = @{}` silently overwrites the `-Settings` argument. Pick a distinct local name (`$requestBody`, `$mergedSettings`). Assigning to a parameter on purpose (normalising its value) is fine, but write it with the parameter's PascalCase name so the intent is visible.
- Declare constants with `Set-Variable -Name maxPageSize -Value 100 -Option ReadOnly -Scope Script`, or use `-Option Constant` for true constants.
- File names are case-sensitive on Linux and macOS. Make file names and every reference to them match exactly.

```powershell
enum IssueState {
    open
    closed
    inProgress
}

class GitHubIssue {
    [int]$number
    [string]$issueTitle
    [IssueState]$state
}
```

## Approved Verbs

- **Always** use a verb from `Get-Verb`. Run `Get-Verb` (or `Get-Verb -Verb <name>`) to check before you name a function.
- Pick the verb by meaning, not by which one is closest:
  - `Get` reads, `Set` overwrites, `New` creates, `Remove` deletes, `Add` appends, `Update` modifies part of something
  - `Invoke` performs an action, `ConvertTo` / `ConvertFrom` transform data
  - `Test` returns `[bool]`, `Assert` throws on failure
- Unapproved verbs are never allowed, even in private functions: `Fetch`, `Create`, `Delete`, `Make`, `Check`, `Validate`, `Load`, `Process`, `Execute`, `Retrieve`.
- `PSUseApprovedVerbs` and `PSUseSingularNouns` must pass with zero suppressions.
