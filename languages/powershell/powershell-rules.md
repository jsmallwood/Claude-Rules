---
paths:
  - "**/*.ps1"
  - "**/*.psm1"
  - "**/*.psd1"
---

# PowerShell Rules (All Versions)

These rules apply to every PowerShell file, whatever version it targets. Version-specific rules are in
`powershell-5.1-rules.md` and `powershell-7-rules.md`. Naming rules are in `powershell-naming-rules.md`. Tool-specific rules are in
`powershell-testing-rules.md` (Pester), `powershell-documentation-rules.md` (PlatyPS),
`powershell-logging-rules.md` (PSFramework Logging), and `powershell-configuration-rules.md` (PSFramework Configuration).
Code-structure rules are in `powershell-functions-rules.md`, `powershell-classes-rules.md`, `powershell-modules-rules.md`,
`powershell-scripts-rules.md`, `powershell-error-handling-rules.md`, and `powershell-error-handling-retry-rules.md`.

## Required Toolchain

| Concern | Tool | Rule |
| --- | --- | --- |
| Static analysis | [PSScriptAnalyzer](https://learn.microsoft.com/en-us/powershell/utility-modules/psscriptanalyzer/overview?view=ps-modules) | **Always** run. Code is not done until it passes. |
| Testing & mocking | [Pester 6.2+](https://pester.dev) | All tests and mocks use Pester. No other test frameworks. |
| Help documentation | [platyPS 0.14.x](https://github.com/PowerShell/platyPS) (≥ 0.14.1, required by PowerShellBuild) | Generates all `Get-Help` content for modules. |
| Build | [PowerShellBuild](https://github.com/psake/PowerShellBuild) | Builds, tests, documents, and publishes modules. |
| Logging | [PSFramework Logging](https://psframework.org/docs/category/logging) | All diagnostic, progress, and error messages go through PSFramework. |
| Configuration | [PSFramework Configuration](https://psframework.org/docs/category/configuration) | Use it whenever settings are needed. Don't build your own config system. |

## Determine the Target Version First

Before you write or change code, find out which PowerShell version the code targets:

1. `#Requires -Version` at the top of scripts
2. `PowerShellVersion` and `CompatiblePSEditions` in the module manifest (`.psd1`)
3. CI matrix or build scripts

- `Desktop` / 5.1 only: follow `powershell-5.1-rules.md`.
- `Core` / 7.x only: follow `powershell-7-rules.md`.
- Both editions: write to the **5.1 language subset**, follow both files, and test on both versions.
- If you can't tell, ask the user. Don't guess.

## Naming Conventions & Approved Verbs

See `powershell-naming-rules.md`. In short: camelCase variables, PascalCase parameters and types,
`<ApprovedVerb>-<Prefix><SingularNoun>` functions, and only verbs from `Get-Verb`.

## Functions, Classes, Modules & Scripts

Detailed rules live in dedicated files:

- `powershell-functions-rules.md`: advanced function template, parameters, pipeline, ShouldProcess, scope
- `powershell-classes-rules.md`: classes and enums, load order, exporting types
- `powershell-modules-rules.md`: directory structure, manifest, PowerShellBuild build/test/publish
- `powershell-scripts-rules.md`: standalone script structure, exit codes, logging setup

## Style

- Use One True Brace Style (OTBS): the opening brace goes on the same line. Indent with 4 spaces, never tabs.
- Keep lines to **115 characters** or fewer. Use splatting or line breaks after a pipe, comma, or operator to stay within it.
- Write full cmdlet and parameter names. **No aliases** in scripts (`ForEach-Object` not `%`, `Where-Object` not `?`, `Get-ChildItem` not `ls`/`gci`).
- Use named parameters for anything beyond the first obvious positional argument.
- Use splatting when a call has more than 3–4 parameters:

  ```powershell
  $restParams = @{
      Uri         = $uri
      Method      = 'Post'
      Headers     = $apiHeader
      Body        = $body | ConvertTo-Json -Depth 10
      ContentType = 'application/json'
      ErrorAction = 'Stop'
  }
  Invoke-RestMethod @restParams
  ```

- Use single quotes for literal strings and double quotes only when you need interpolation.
- Put `$null` on the left of comparisons: `if ($null -eq $value)`.
- Don't use backtick line continuation. Use splatting, or break lines after a pipe, comma, or operator.
- Don't use `Write-Host` for diagnostics. Use `Write-PSFMessage` (see `powershell-logging-rules.md`).
- Run `Invoke-Formatter` (from PSScriptAnalyzer) to keep formatting consistent.

## Error Handling

See `powershell-error-handling-rules.md` (try/catch, Stop-PSFFunction, error records, native and HTTP errors) and
`powershell-error-handling-retry-rules.md` (retry with exponential backoff and jitter for transient failures).
Never leave a `catch {}` empty.

## Performance

From [PowerShell scripting performance considerations](https://learn.microsoft.com/en-us/powershell/scripting/dev-cross-plat/performance/script-authoring-considerations?view=powershell-7.6).
Apply these in loops and on large data. For small, one-off operations, prefer the most readable form.

- Suppress output with `$null = ...`, not `| Out-Null` or `[void]` in hot paths.
- Collect loop results by assigning the loop output directly (`$items = foreach (...) { ... }`). It's the fastest pattern on every version. When items are added conditionally, use `[System.Collections.Generic.List[T]]` and `.Add()`.
- Array `+=` in a loop is O(n²) on **5.1 and 7.4**; 7.5 optimised it. Flag it when the code targets 5.1 or 7.4, or when the loop runs over an unbounded collection on any version. Don't flag a small, bounded loop in 7.5+-only code.
- Never use `[System.Collections.ArrayList]`. It's a non-generic legacy type that boxes values and needs `$null =` on every `.Add()`. Use `List[T]`.
- Don't build strings with `+=` in a loop. Use `-join` or `[System.Text.StringBuilder]`.
- For lookups by key across large collections, build a hashtable once instead of calling `Where-Object` inside a loop.
- Read large files with `[System.IO.StreamReader]` / `[IO.File]::ReadLines()` instead of `Get-Content` into memory.
- Filter as early as possible: use a cmdlet's `-Filter` parameter before `Where-Object`.
- Don't call a function once per item in a tight loop when it can accept the whole collection through the pipeline.
- Use `ForEach-Object -Parallel` / thread jobs only for I/O-bound work where the overhead pays off (see `powershell-7-rules.md` and [parallel execution](https://learn.microsoft.com/en-us/powershell/scripting/dev-cross-plat/performance/parallel-execution?view=powershell-7.6)).
- Measure with `Measure-Command` before and after an optimisation. Don't optimise on guesswork.

## Security

- Never hard-code secrets, tokens, or passwords. Accept a `[PSCredential]` or `[SecureString]`, or read them through `Microsoft.PowerShell.SecretManagement`.
- Never use `Invoke-Expression` on input that isn't fully trusted.
- Don't use `ConvertTo-SecureString -AsPlainText` with literal values.
- Don't log secrets. Don't pass credentials as `-Target` to `Write-PSFMessage`.

## Module Layout

See `powershell-modules-rules.md` for the required directory structure and the PowerShellBuild workflow.

## Static Analysis — PSScriptAnalyzer (Mandatory)

Use PSScriptAnalyzer 1.25 or later. Run it on every change before you call the work finished:

```powershell
Invoke-ScriptAnalyzer -Path . -Recurse -Settings ./PSScriptAnalyzerSettings.psd1
```

- The result must have **zero** `Error` and `Warning` findings.
- Keep a `PSScriptAnalyzerSettings.psd1` in the repository root. Don't rely on defaults that differ from machine to machine.
- Configure the compatibility rules (`PSUseCompatibleSyntax`, `PSUseCompatibleCommands`) for the target versions (see the version files).
- Suppress a rule only on a specific function, using `[Diagnostics.CodeAnalysis.SuppressMessageAttribute('RuleName', '', Justification = '...')]`, and always give a real justification. Never suppress a rule globally just to get a clean run.
- Run PSScriptAnalyzer in CI and fail the build when it reports findings.
- If you can't run PSScriptAnalyzer, say so explicitly. Never report code as passing analysis when it wasn't run.

Baseline settings file:

```powershell
@{
    Severity     = @('Error', 'Warning', 'Information')
    IncludeDefaultRules = $true
    Rules        = @{
        PSUseConsistentIndentation = @{ Enable = $true; IndentationSize = 4; Kind = 'space' }
        PSUseConsistentWhitespace  = @{ Enable = $true }
        PSPlaceOpenBrace           = @{ Enable = $true; OnSameLine = $true }
        PSPlaceCloseBrace          = @{ Enable = $true; NewLineAfter = $true }
        PSAlignAssignmentStatement = @{ Enable = $true; CheckHashtable = $true }
        PSUseCorrectCasing         = @{ Enable = $true }
        PSAvoidUsingCmdletAliases  = @{ Enable = $true }
        PSAvoidLongLines           = @{ Enable = $true; MaximumLineLength = 115 }
        PSUseCompatibleSyntax      = @{ Enable = $true; TargetVersions = @('5.1', '7.4') } # adjust per target
    }
}
```

PSScriptAnalyzer also ships a `CodeFormattingOTBS` preset (`Invoke-ScriptAnalyzer -Settings CodeFormattingOTBS`)
that covers brace style, indentation, spacing, alignment, and casing. It's a quick check without a settings file,
but it doesn't include the line-length rule, so the repository settings file above stays the source of truth.

## Definition of Done

1. PSScriptAnalyzer reports zero findings (Error/Warning).
2. Pester tests exist and pass on every targeted PowerShell version.
3. PlatyPS markdown help is created or updated for every public function you added or changed, with no `{{ Fill in` placeholders, and the function carries `.EXTERNALHELP`.
4. Logging uses `Write-PSFMessage` / `Stop-PSFFunction`. There's no `Write-Host` for diagnostics.
5. Any new settings are declared with `Set-PSFConfig -Initialize`.
6. Modules build and test through PowerShellBuild (`./build.ps1 -Task Test`).

## References

- [ai-powershell-standards](https://github.com/fadwen/ai-powershell-standards/tree/main)
- [What is PowerShell?](https://learn.microsoft.com/en-us/powershell/scripting/overview?view=powershell-7.6)
- [Writing portable modules](https://learn.microsoft.com/en-us/powershell/scripting/dev-cross-plat/writing-portable-modules?view=powershell-7.6)
- [PowerShell scripting performance considerations](https://learn.microsoft.com/en-us/powershell/scripting/dev-cross-plat/performance/script-authoring-considerations?view=powershell-7.6)
- [PowerShell module authoring considerations](https://learn.microsoft.com/en-us/powershell/scripting/dev-cross-plat/performance/module-authoring-considerations?view=powershell-7.6)
- [Optimize performance using parallel execution](https://learn.microsoft.com/en-us/powershell/scripting/dev-cross-plat/performance/parallel-execution?view=powershell-7.6)
- [PSScriptAnalyzer](https://learn.microsoft.com/en-us/powershell/utility-modules/psscriptanalyzer/overview?view=ps-modules)
- [Pester](https://pester.dev) / [platyPS](https://github.com/PowerShell/platyPS)
- [PSFramework Logging](https://psframework.org/docs/category/logging) / [PSFramework Configuration](https://psframework.org/docs/category/configuration)
- [PowerShellBuild](https://github.com/psake/PowerShellBuild)
