---
paths:
  - "**/*.ps1"
  - "**/*.psm1"
---

# PowerShell Error Handling Rules

These rules cover error handling **without retry**. For transient failures that should be retried with backoff, also
follow `powershell-error-handling-retry-rules.md`. Logging of errors uses PSFramework (`powershell-logging-rules.md`).

## Terminating vs Non-Terminating

- Most cmdlet errors are **non-terminating**: they write to the error stream and execution continues, so `try` / `catch` **doesn't catch them**.
- Inside a `try`, add `-ErrorAction Stop` to every call that can fail, so its errors become catchable.
- Don't set `$ErrorActionPreference = 'Stop'` inside functions. Scripts set it at script scope (see `powershell-scripts-rules.md`).
- Use `-ErrorAction SilentlyContinue` / `Ignore` only when a missing result is expected **and** you check for it straight away. Prefer an explicit test (`Test-Path`, `Get-Command -ErrorAction Ignore` followed by a `$null` check).

## try / catch / finally

```powershell
try {
    $content = Get-Content -Path $configPath -Raw -ErrorAction Stop
    $settings = $content | ConvertFrom-Json -ErrorAction Stop
}
catch [System.Management.Automation.ItemNotFoundException] {
    Stop-PSFFunction -Message "Configuration file not found: $configPath" -ErrorRecord $_ -Target $configPath -EnableException $EnableException
    return
}
catch [System.ArgumentException] {
    Stop-PSFFunction -Message "Configuration file is not valid JSON: $configPath" -ErrorRecord $_ -Target $configPath -EnableException $EnableException
    return
}
catch {
    Stop-PSFFunction -Message "Unexpected error reading $configPath" -ErrorRecord $_ -Target $configPath -EnableException $EnableException
    return
}
```

- Catch **specific** exception types first and the generic `catch` last. Catch only what you can handle or describe better.
- Inside `catch`, `$_` is the `ErrorRecord`. The exception is `$_.Exception`, and the root cause is often `$_.Exception.InnerException`.
- Copy `$_` into a variable (`$errorRecord = $_`) before calling other commands in the `catch`, because `$_` can be overwritten.
- Use `finally` for cleanup: disposing objects, closing connections, `Pop-Location`, removing temporary files. It runs on success, on failure, and when the user presses Ctrl+C.
- Keep `try` blocks small. Wrap only the statements whose failure you're handling.
- **Never** leave a `catch {}` empty, and never catch just to log and carry on as if nothing happened, unless continuing is the documented behavior.
- Don't use `trap`.

## Reporting Errors from Functions

| Situation | Use |
| --- | --- |
| Public function: operation failed (the standard pattern) | `Stop-PSFFunction -Message ... -ErrorRecord $_ -EnableException $EnableException` (+ `-Continue` in loops / `process`) |
| One pipeline item failed, others should continue | `Stop-PSFFunction ... -Continue` |
| Private function: let the caller decide | Rethrow with a bare `throw` inside `catch`, or let the exception propagate |
| Class method | `throw [SpecificException]::new(...)` (see `powershell-classes-rules.md`) |
| Invalid argument combination detected in the body | `$PSCmdlet.ThrowTerminatingError($errorRecord)` with a specific `ErrorCategory` |

- Rethrow with a bare `throw` (inside `catch`) to keep the original stack trace. Don't use `throw $_.Exception.Message`, which loses the type and the stack.
- When you wrap an error with more context, keep the original as the inner exception:

```powershell
catch {
    $exception = [System.InvalidOperationException]::new("Failed to load issue $IssueNumber from $RepoOwner/$RepoName", $_.Exception)
    $errorRecord = [System.Management.Automation.ErrorRecord]::new(
        $exception,
        'GtIssueLoadFailed',
        [System.Management.Automation.ErrorCategory]::ReadError,
        $IssueNumber
    )
    $PSCmdlet.ThrowTerminatingError($errorRecord)
}
```

- Error IDs are PascalCase, start with the module prefix, and are stable (`GtIssueNotFound`, `GtApiRateLimited`). Tests and callers may depend on them.
- Pick the `ErrorCategory` that fits (`ObjectNotFound`, `InvalidArgument`, `PermissionDenied`, `ConnectionError`, `ReadError`, `WriteError`, `InvalidOperation`, `OperationTimeout`).
- Error messages say **what** failed, **which** object, and, when known, **what to do about it**. Never include secrets or tokens.

## Native Commands

- Native executables don't throw. Check `$LASTEXITCODE` straight after each call:

```powershell
& git fetch --prune 2>&1 | ForEach-Object { Write-PSFMessage -Level Verbose -Message "$_" }
if ($LASTEXITCODE -ne 0) {
    throw [System.InvalidOperationException]::new("git fetch failed with exit code $LASTEXITCODE")
}
```

- In PowerShell 7.4+ scripts, `$PSNativeCommandUseErrorActionPreference = $true` can be used instead (see `powershell-7-rules.md`).
- Use `$?` and `$LASTEXITCODE` only for native commands, never for cmdlets.

## HTTP Errors

- Read the HTTP status from the exception's response:
  - Windows PowerShell 5.1: `[System.Net.WebException]`, with `$_.Exception.Response.StatusCode` (`HttpWebResponse`)
  - PowerShell 7: `[Microsoft.PowerShell.Commands.HttpResponseException]`, with `$_.Exception.Response.StatusCode` (`HttpResponseMessage`)
- Map well-known status codes to clear messages and error categories: 401/403 → `PermissionDenied`, 404 → `ObjectNotFound`, 422 → `InvalidData`.
- Decide whether the error is **transient** (could succeed if tried again) or **permanent**. Transient errors go to the retry rules. Permanent errors fail straight away.

## Classifying Errors

| Permanent: fail immediately, never retry | Transient: candidates for retry (see the retry rules) |
| --- | --- |
| Validation errors, `ArgumentException` | HTTP 408, 429, 500, 502, 503, 504 |
| HTTP 400, 401, 403, 404, 405, 409, 422 | Connection reset / refused, DNS timeouts |
| File or item not found, access denied | Request timeouts |
| Parse errors, wrong input format | Locked files, deadlocks (domain-specific) |

## Testing

- Every function has tests for its failure paths:
  - `{ Get-GtIssue ... -EnableException } | Should-Throw -ExceptionType ([System.InvalidOperationException])`
  - `{ ... } | Should-Throw -FullyQualifiedErrorId 'GtIssueNotFound*'`
- Without `-EnableException`, check that no exception is thrown and that a warning was written (`Should-Invoke Write-PSFMessage -ParameterFilter { $Level -eq 'Warning' }`, or check `-WarningVariable`).
- Check that `finally` cleanup runs (for example, a temporary file is removed after a failure).
