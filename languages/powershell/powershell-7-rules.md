---
paths:
  - "**/*.ps1"
  - "**/*.psm1"
  - "**/*.psd1"
---

# PowerShell 7 Rules

**Applies when** the code targets PowerShell 7.x only (`#Requires -Version 7.x`, manifest
`CompatiblePSEditions = 'Core'`). If the code must also run on Windows PowerShell 5.1, follow
`powershell-5.1-rules.md` for language features, and use only the parts of this file that don't need 7-only syntax.

All other rules files in this folder (`powershell-*-rules.md`) also apply.

## Declaring the Target

Support lifecycle (per the ai-powershell-standards repository, verified 2026-08-01; recheck Microsoft's
[support lifecycle](https://learn.microsoft.com/en-us/powershell/scripting/install/powershell-support-lifecycle) when it matters):

| Version | Type | End of support | Runtime |
| --- | --- | --- | --- |
| **7.6** | **LTS: current target** | 14-Nov-2028 | .NET 10 |
| 7.5 | Stable | 10-Nov-2026 | .NET 9 |
| 7.4 | LTS (previous) | 10-Nov-2026 | .NET 8 |
| 7.0–7.3, 6.x | Retired | — | — |

- Target **7.6** for all new work unless the project says otherwise. Projects pinned to 7.4 or 7.5 need a planned upgrade.
- Never declare `#Requires -Version 7.0` or target 6.x / 7.0–7.3 in code or CI matrices. If you truly mean "any supported 7.x", the floor is `7.4` until 10-Nov-2026 and `7.6` after it.
- Use `7.4` as the minimum only after confirming the code uses none of the 7.5+/7.6+ features below.
- Scripts: `#Requires -Version 7.6`
- Module manifest:

  ```powershell
  PowerShellVersion    = '7.6'
  CompatiblePSEditions = @('Core')
  RequiredModules      = @('PSFramework')
  ```

- PSScriptAnalyzer: `PSUseCompatibleSyntax = @{ Enable = $true; TargetVersions = @('7.4') }` (the oldest supported syntax target). Also turn on `PSUseCompatibleCommands` with profiles for each OS you support.

## Version-Dependent Features

Don't use these unless the declared minimum version supports them.

| Minimum | Feature |
| --- | --- |
| 7.5 | `ConvertTo-CliXml` / `ConvertFrom-CliXml` (no temp file), `ConvertFrom-Json -DateKind` (use `Utc` or `Offset` for audit data), `Test-Json -IgnoreComments` / `-AllowTrailingCommas`, `New-Guid -Empty`, `Get-Process -IncludeUserName` without admin |
| 7.6 | `Get-Command -ExcludeModule`, `Register-ArgumentCompleter -NativeFallback`, `Get-Clipboard -Delimiter`, `Join-Path` with several `-ChildPath` segments in one call |

## Breaking Changes to Code Around

- **7.6:** the `ThreadJob` module is renamed `Microsoft.PowerShell.ThreadJob`. Call `Start-ThreadJob` without a module qualifier, and declare `Microsoft.PowerShell.ThreadJob` (not `ThreadJob`) in `RequiredModules`.
- **7.6:** `[WildcardPattern]::Escape()` now escapes lone backticks correctly. Remove any double-escaping workaround.
- **7.5:** `Test-Path -OlderThan` and `-NewerThan` now both apply when used together (before, `-OlderThan` was silently ignored).
- **7.5:** `ConvertTo-Json` serializes `BigInteger` as a number, not an object.

## Detecting the Version

- Compare the full version, never only `.Major`: `if ($PSVersionTable.PSVersion -ge [version]'7.6') { }`. `Major -ge 7` is true for every 7.x.
- Prefer detecting the **capability** when one exists: `(Get-Command Invoke-WebRequest).Parameters.ContainsKey('SkipHttpErrorCheck')`. Detect once and cache it in `$script:` scope.
- When diagnosing platform behaviour, log `[System.Runtime.InteropServices.RuntimeInformation]::FrameworkDescription`.

## Modern Language Features

You may use these when they make code clearer, and they must stay readable:

- Ternary: `$label = $isOpen ? 'open' : 'closed'`. Keep it to simple expressions and never nest ternaries.
- Null-coalescing: `$PageSize = $PageSize ?? (Get-PSFConfigValue -FullName 'GitHubTools.Api.PageSize')`
- Null-conditional: `${issue}?.title`. You need the braces because `?` is a valid variable-name character.
- Pipeline chains: `&&` and `||` go in interactive or build scripts only. In module code, use `try` / `catch`.
- `clean {}` block (7.3+): use it for cleanup that has to run even when the pipeline stops early.
- `Get-Error` for diagnostics, and `Test-Json` for schema validation.
- `ConvertFrom-Json -AsHashtable` when you need dictionary semantics or keys that differ only by case.

## Parallelism

- `ForEach-Object -Parallel` is allowed for I/O-bound work. Always set `-ThrottleLimit`.
- Variables from outside the block must be passed with `$using:`. Runspaces don't share state, so treat `$using:` values as read-only.
- Modules aren't automatically loaded into parallel runspaces the way you might expect. Import what the block needs inside the block, including PSFramework if it logs.
- PSFramework logging works from parallel runspaces. Include `-Target` in messages so the interleaved entries can be traced.
- Use `-AsJob` / `Start-ThreadJob` for long-running background work. Don't use parallelism for CPU-light loops over small collections, where the overhead costs more than it saves.

## Cross-Platform

PowerShell 7 code is assumed to run on Windows, Linux, and macOS unless it's documented as Windows-only.

- Build paths with `Join-Path` (several child segments in one call need 7.6) or `[IO.Path]::Combine()`. Never hard-code `\` or `/` separators.
- Use `$IsWindows`, `$IsLinux`, and `$IsMacOS` for platform branches, and log the branch taken at `Verbose` level.
- File names are case-sensitive on Linux. Make file names and their references match exactly.
- Get special locations from `[Environment]::GetFolderPath()` or `$HOME`, not `$env:USERPROFILE` or `C:\`.
- Windows-only functions must say so in their help, and check `$IsWindows` and stop with `Stop-PSFFunction` when it's false.
- Don't use `Import-Module -UseWindowsPowerShell` (the Windows compatibility layer) in modules. If it's truly unavoidable, document it.
- WMI cmdlets don't exist in 7. Use CIM cmdlets (Windows only).

## Encoding

- The default encoding is UTF-8 without BOM (`utf8NoBOM`). Still pass `-Encoding` explicitly on file-writing cmdlets so the intent is clear.
- If the file must be read by Windows PowerShell 5.1 and contains non-ASCII text, write it as `utf8BOM`.

## Web & Native Commands

- Use the native `Invoke-RestMethod` features: `-SkipHttpErrorCheck`, `-StatusCodeVariable`, `-ResponseHeadersVariable`, `-Authentication Bearer -Token <SecureString>`, `-MaximumRetryCount`, `-RetryIntervalSec`, `-FollowRelLink` (for paging).
- Don't pass `-UseBasicParsing`. It's ignored in 7 and only adds noise.
- In PowerShell 7.4+, set `$PSNativeCommandUseErrorActionPreference = $true` in scripts that call native executables, so failing native commands respect `$ErrorActionPreference`. In earlier versions, check `$LASTEXITCODE` explicitly.
- `ConvertTo-Json` still defaults to `-Depth 2`. Always pass `-Depth`.

## .NET

- PowerShell 7 runs on modern .NET. You can use modern APIs (for example `System.Text.Json`), but note the minimum PowerShell version they need.
- Don't load assemblies built for .NET Framework that have no .NET Standard / .NET equivalent.

## Tooling on 7

- Install modules with `Install-PSResource` (Microsoft.PowerShell.PSResourceGet, 1.2.0 in-box in 7.6). Use `Install-Module` only where PSResourceGet isn't available.
- Pester 6.2+, PSScriptAnalyzer 1.25+, PSFramework, and platyPS 0.14.x all support PowerShell 7.
- Run CI on every supported OS (for example a matrix of `windows-latest`, `ubuntu-latest`, and `macos-latest` with `pwsh`).
