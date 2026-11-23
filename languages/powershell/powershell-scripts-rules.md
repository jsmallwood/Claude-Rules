---
paths:
  - "**/*.ps1"
---

# PowerShell Script Rules

These rules apply to standalone `.ps1` scripts: automation, scheduled jobs, CI steps, and tools that are run directly.
They don't apply to module source files in `Public/`, `Private/`, `Classes/`, `Enum/`, or `Configurations/`, to
`*.Tests.ps1`, or to `psakeFile.ps1`.

## When to Write a Script vs a Module

- A script is a thin **entry point**: it parses parameters, sets up logging, calls functions, and sets the exit code.
- When logic is reused across scripts, or grows beyond about 200 lines, move it into a module (see `powershell-modules-rules.md`) and have the script import that module.
- Name script files with an approved verb and a PascalCase noun: `Invoke-NightlyIssueSync.ps1`, `Export-GtIssueReport.ps1`.

## Template

```powershell
<#
.SYNOPSIS
    Synchronises open GitHub issues to the reporting database.

.DESCRIPTION
    Reads every open issue in the given repository and upserts it into the reporting database.
    Logs to the PSFramework logfile provider.

.PARAMETER RepoOwner
    Owner of the GitHub repository.

.PARAMETER RepoName
    Name of the GitHub repository.

.PARAMETER LogPath
    Folder for log files. Defaults to a 'logs' folder next to the script.

.EXAMPLE
    ./Invoke-NightlyIssueSync.ps1 -RepoOwner 'octo' -RepoName 'demo' -WhatIf

    Shows what would be synchronised without writing to the database.
#>
#Requires -Version 5.1
#Requires -Modules PSFramework, GitHubTools

[CmdletBinding(SupportsShouldProcess)]
param (
    [Parameter(Mandatory)]
    [string]$RepoOwner,

    [Parameter(Mandatory)]
    [string]$RepoName,

    [ValidateNotNullOrEmpty()]
    [string]$LogPath = (Join-Path -Path $PSScriptRoot -ChildPath 'logs')
)

Set-StrictMode -Version Latest
$ErrorActionPreference = 'Stop'

$logParams = @{
    Name         = 'logfile'
    InstanceName = 'NightlyIssueSync'
    FilePath     = Join-Path -Path $LogPath -ChildPath 'NightlyIssueSync-%Date%.log'
    Enabled      = $true
}
Set-PSFLoggingProvider @logParams

$exitCode = 0
try {
    Write-PSFMessage -Level Host -Message "Starting issue sync for $RepoOwner/$RepoName"

    $issues = Get-GtIssue -RepoOwner $RepoOwner -RepoName $RepoName -State 'open' -EnableException
    foreach ($issue in $issues) {
        if ($PSCmdlet.ShouldProcess("$RepoOwner/$RepoName#$($issue.number)", 'Sync issue')) {
            Sync-GtIssueRecord -InputObject $issue -EnableException
        }
    }

    Write-PSFMessage -Level Host -Message "Synchronised $(@($issues).Count) issue(s)"
}
catch {
    Write-PSFMessage -Level Critical -Message 'Issue sync failed' -ErrorRecord $_
    $exitCode = 1
}
finally {
    Wait-PSFMessage
}

exit $exitCode
```

## Structure (in this order)

1. Comment-based help: `.SYNOPSIS`, `.DESCRIPTION`, a `.PARAMETER` for each parameter, and at least one `.EXAMPLE`. Scripts use comment-based help, not PlatyPS (see `powershell-documentation-rules.md`).
2. `#Requires` statements: `-Version` (always), `-Modules` (always, including `PSFramework`), and `-RunAsAdministrator` only when it's really needed.
3. `[CmdletBinding()]` and a `param ()` block. Add `SupportsShouldProcess` if the script changes anything.
4. `Set-StrictMode -Version Latest` and `$ErrorActionPreference = 'Stop'`. These are set at script scope only, never inside module functions.
5. Logging provider setup (see `powershell-logging-rules.md`).
6. Main logic in a single top-level `try` / `catch` / `finally`.
7. `finally` calls `Wait-PSFMessage` and releases resources.
8. `exit $exitCode` as the last statement.

## Rules

- Parameters follow the function rules: PascalCase, typed, and validated with attributes (see `powershell-functions-rules.md`).
- **Exit codes:** `0` means success and non-zero means failure. Document any specific codes in `.DESCRIPTION`. `exit` is used **only** at the end of a script, never inside functions or modules.
- **Paths:** resolve paths relative to the script with `$PSScriptRoot`. Never depend on the caller's current directory. If you must change location, use `Push-Location` / `Pop-Location` in `finally`.
- **No interactive prompts** in automation scripts. Don't use `Read-Host`, `Get-Credential` without a parameter, or `$Host.UI.PromptForChoice`. Accept everything through parameters, PSFramework configuration, or SecretManagement.
- **No global side effects:** don't change `$global:` variables, `$env:PSModulePath`, the execution policy, or the user profile.
- Don't define functions in a script that duplicate functions in a module. If a helper function grows beyond a few lines, move it into a module.
- Settings beyond a few parameters come from PSFramework Configuration (`Import-PSFConfig` for a settings file), not from hand-parsed JSON or INI files (see `powershell-configuration-rules.md`).
- Use `-EnableException` when calling module functions, so failures reach the script's top-level `catch`.
- Scheduled or CI execution: `pwsh -NoProfile -NonInteractive -File ./Invoke-NightlyIssueSync.ps1 ...` (or `powershell.exe` for 5.1).
- In environments that enforce `AllSigned`, sign scripts with `Set-AuthenticodeSignature`, and re-sign after every change.

## Testability

- Put the logic in module functions and test those with Pester. The script itself should be too thin to need many tests.
- If a script has to contain testable functions, guard the main body so Pester can dot-source the file without running it:

```powershell
if ($MyInvocation.InvocationName -ne '.') {
    Invoke-Main
}
```

- Test scripts end to end with Pester by running them in a child process and checking the exit code (`& pwsh -NoProfile -File $scriptPath ...; $LASTEXITCODE | Should-Be 0`), with external systems mocked or pointed at test doubles.
- Run PSScriptAnalyzer on every script.
