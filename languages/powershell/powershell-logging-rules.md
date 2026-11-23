---
paths:
  - "**/*.ps1"
  - "**/*.psm1"
  - "**/*.psd1"
---

# PowerShell Logging Rules — PSFramework

Reference: [PSFramework — Logging](https://psframework.org/docs/category/logging)

[PSFramework](https://psframework.org) is the standard library for logging. Declare it in the module manifest's
`RequiredModules`. Scripts declare it with `#Requires -Modules PSFramework`.

## Use PSFramework, not the built-in write cmdlets

| Instead of | Use |
| --- | --- |
| `Write-Verbose` | `Write-PSFMessage -Level Verbose` |
| `Write-Debug` | `Write-PSFMessage -Level Debug` |
| `Write-Warning` | `Write-PSFMessage -Level Warning` |
| `Write-Host` (status or diagnostic text) | `Write-PSFMessage -Level Host` (or `Important`) |
| `Write-Error` / `throw` in public functions | `Stop-PSFFunction` (private functions and class methods still `throw`; see `powershell-error-handling-rules.md`) |

Data is still returned on the output stream. PSFramework replaces diagnostic messages, not output.

## Write-PSFMessage

```powershell
Write-PSFMessage -Level Verbose -Message "Requesting page $pageNumber for $RepoOwner/$RepoName" -Tag 'api', 'paging' -Target $RepoName
Write-PSFMessage -Level Warning -Message "Rate limit low: $remaining requests left" -Tag 'api', 'ratelimit'
Write-PSFMessage -Level Important -Message "Created issue #$($issue.number)" -Target $issue.number
```

- Always pass `-Message`. Add `-Tag` (lowercase, consistent within a module) so logs can be filtered.
- Pass `-Target` with the object being worked on, so log entries can be traced to it.
- Pass `-ErrorRecord $_` when you log from inside a `catch`.
- Choose levels on purpose:
  - `Verbose`: normal operational detail
  - `Debug`: developer tracing
  - `Host` / `Important`: messages the user should see
  - `Warning`: something recoverable went wrong
  - `Critical`: severe failures
- Never put secrets, tokens, or raw PII in `-Message`, `-Target`, or tags.
- To support localized messages, use `-String` with strings registered through `Import-PSFLocalizedString`.

## Error handling with Stop-PSFFunction

Every public function takes an `-EnableException` switch. By default it writes a friendly warning; with the switch
set, it throws a terminating exception.

```powershell
process {
    foreach ($issueNumber in $IssueNumbers) {
        try {
            Remove-GtIssueInternal -IssueNumber $issueNumber -ErrorAction Stop
        }
        catch {
            Stop-PSFFunction -Message "Failed to remove issue $issueNumber" -ErrorRecord $_ -Target $issueNumber -EnableException $EnableException -Continue
        }
    }
}
```

- Use `-Continue` inside loops or `process` blocks to skip to the next item.
- Without `-Continue`, call `return` right after `Stop-PSFFunction`. Or check `if (Test-PSFFunctionInterrupt) { return }` at the start of `process {}` when `Stop-PSFFunction` was called in `begin {}`.

## Logging providers

- Configure providers in scripts and entry points, **never** in reusable module functions. Modules write messages, and the caller decides where they go.

```powershell
$logParams = @{
    Name         = 'logfile'
    InstanceName = 'GitHubTools'
    FilePath     = 'C:\Logs\GitHubTools-%Date%.log'
    Enabled      = $true
}
Set-PSFLoggingProvider @logParams
```

- At the end of a script that uses file or remote logging providers, call `Wait-PSFMessage` so the log queue is flushed before the process exits.
- Use `Get-PSFMessage` while debugging to look at recent messages in the session.

## Testing

- Mock `Write-PSFMessage` only when you need to assert that something was logged. Leave it unmocked otherwise.
