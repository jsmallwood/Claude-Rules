---
paths:
  - "**/*.ps1"
  - "**/*.psm1"
---

# PowerShell Error Handling Rules — Retry & Backoff

These rules cover **retrying transient failures** with exponential backoff and jitter. Everything in
`powershell-error-handling-rules.md` still applies. Retry is an extra layer, not a replacement for proper error
handling.

## When to Retry

Retry only when **all** of these are true:

1. The error is **transient**: HTTP 408, 429, 500, 502, 503, 504, a connection failure, or a timeout (see the classification table in `powershell-error-handling-rules.md`).
2. The operation is **idempotent**: GET, PUT, DELETE, or a POST that carries an idempotency key. Never automatically retry a non-idempotent call that creates resources or sends messages, because you might create duplicates.
3. The caller can afford to wait: the total time across retries stays within a documented budget.

Never retry:

- Validation errors, HTTP 400/401/403/404/409/422, or authentication failures
- Errors that `ShouldProcess` or the user declined
- Inside another retry loop. Nested retries multiply the attempt count, so retry at **one** layer only, usually the lowest private HTTP or IO helper.

## Choosing the Mechanism

| Need | Use |
| --- | --- |
| Exponential backoff, jitter, `Retry-After`, transient-only filtering (**the default for HTTP APIs**) | The private `Invoke-<Prefix>WithRetry` helper below |
| Simple fixed-interval retry of a local operation (locked file, service starting up) | `Invoke-PSFProtectedCommand` with `-RetryCount` / `-RetryWait` / `-RetryErrorType` |
| PowerShell 7 `Invoke-RestMethod -MaximumRetryCount -RetryIntervalSec` | **Don't use it for API calls.** It waits a fixed interval and also retries non-transient 4xx responses. |

## Settings (PSFramework Configuration)

Retry limits are configurable. Declare them in `Configurations/` (see `powershell-configuration-rules.md`):

```powershell
Set-PSFConfig -Module 'GitHubTools' -Name 'Retry.MaxAttempts' -Value 5 -Initialize -Validation 'integerpositive' -Description 'Maximum attempts (including the first) for transient failures.'
Set-PSFConfig -Module 'GitHubTools' -Name 'Retry.BaseDelayMilliseconds' -Value 500 -Initialize -Validation 'integerpositive' -Description 'Base delay for exponential backoff.'
Set-PSFConfig -Module 'GitHubTools' -Name 'Retry.MaxDelayMilliseconds' -Value 30000 -Initialize -Validation 'integerpositive' -Description 'Upper bound on a single backoff delay, and the largest Retry-After honoured.'
```

## Backoff Algorithm

- **Exponential backoff with full jitter:** `delay = Random(0, Min(maxDelay, baseDelay × 2^(attempt − 1)))`.
- Jitter is required. Without it, many clients retry at the same moment and overload the service again.
- **Honour `Retry-After`** (common on 429 and 503) when the server sends it, and use it instead of the computed delay. If it's longer than `Retry.MaxDelayMilliseconds`, don't wait. Fail with a clear rate-limit error.
- Cap the number of attempts (`Retry.MaxAttempts`, which includes the first attempt). There's never an unbounded loop.

## Reference Implementation (5.1 and 7 compatible)

`Private/Test-GtTransientError.ps1`:

```powershell
function Test-GtTransientError {
    [CmdletBinding()]
    [OutputType([bool])]
    param (
        [Parameter(Mandatory)]
        [System.Management.Automation.ErrorRecord]$ErrorRecord
    )

    $transientStatusCodes = @(408, 429, 500, 502, 503, 504)
    $exception = $ErrorRecord.Exception

    $responseProperty = $exception.PSObject.Properties['Response']
    if ($null -ne $responseProperty -and $null -ne $responseProperty.Value) {
        return ($transientStatusCodes -contains [int]$responseProperty.Value.StatusCode)
    }

    # No HTTP response: classify network-level failures. Type names are compared as strings
    # because System.Net.Http types are not loaded by default in Windows PowerShell 5.1.
    $exceptionTypeName = $exception.GetType().FullName
    if ($exceptionTypeName -eq 'System.Net.WebException') {
        $transientWebStatuses = @('Timeout', 'ConnectFailure', 'ConnectionClosed', 'KeepAliveFailure', 'ReceiveFailure', 'SendFailure', 'PipelineFailure')
        return ($transientWebStatuses -contains $exception.Status.ToString())
    }

    $transientExceptionTypes = @(
        'System.Net.Http.HttpRequestException'
        'System.Threading.Tasks.TaskCanceledException'
        'System.TimeoutException'
    )
    return ($transientExceptionTypes -contains $exceptionTypeName)
}
```

`Private/Get-GtRetryAfterMillisecond.ps1`:

```powershell
function Get-GtRetryAfterMillisecond {
    [CmdletBinding()]
    [OutputType([int])]
    param (
        [Parameter(Mandatory)]
        [System.Management.Automation.ErrorRecord]$ErrorRecord
    )

    $responseProperty = $ErrorRecord.Exception.PSObject.Properties['Response']
    if ($null -eq $responseProperty -or $null -eq $responseProperty.Value) { return $null }
    $headers = $responseProperty.Value.Headers

    if ($headers -is [System.Net.WebHeaderCollection]) {
        # Windows PowerShell 5.1 (HttpWebResponse)
        $retryAfterSeconds = 0
        if ([int]::TryParse([string]$headers['Retry-After'], [ref]$retryAfterSeconds)) {
            return $retryAfterSeconds * 1000
        }
        return $null
    }

    # PowerShell 7 (HttpResponseMessage)
    $retryAfter = $headers.RetryAfter
    if ($null -eq $retryAfter) { return $null }
    if ($null -ne $retryAfter.Delta) { return [int]$retryAfter.Delta.TotalMilliseconds }
    if ($null -ne $retryAfter.Date) {
        return [int][math]::Max(0, ($retryAfter.Date.UtcDateTime - [datetime]::UtcNow).TotalMilliseconds)
    }
    return $null
}
```

`Private/Invoke-GtWithRetry.ps1`:

```powershell
function Invoke-GtWithRetry {
    [CmdletBinding()]
    param (
        [Parameter(Mandatory)]
        [scriptblock]$ScriptBlock,

        [Parameter(Mandatory)]
        [string]$OperationName,

        [ValidateRange(1, 20)]
        [int]$MaxAttempts = (Get-PSFConfigValue -FullName 'GitHubTools.Retry.MaxAttempts' -Fallback 5),

        [ValidateRange(1, [int]::MaxValue)]
        [int]$BaseDelayMilliseconds = (Get-PSFConfigValue -FullName 'GitHubTools.Retry.BaseDelayMilliseconds' -Fallback 500),

        [ValidateRange(1, [int]::MaxValue)]
        [int]$MaxDelayMilliseconds = (Get-PSFConfigValue -FullName 'GitHubTools.Retry.MaxDelayMilliseconds' -Fallback 30000)
    )

    $attempt = 0
    while ($true) {
        $attempt++
        try {
            # Capture output so a failed attempt never leaks partial results downstream
            $result = & $ScriptBlock
            if ($attempt -gt 1) {
                Write-PSFMessage -Level Verbose -Message "$OperationName succeeded on attempt $attempt" -Tag 'retry'
            }
            return $result
        }
        catch {
            $errorRecord = $_

            if (-not (Test-GtTransientError -ErrorRecord $errorRecord)) {
                throw
            }
            if ($attempt -ge $MaxAttempts) {
                Write-PSFMessage -Level Warning -Message "$OperationName failed after $attempt attempts; giving up" -ErrorRecord $errorRecord -Tag 'retry'
                throw
            }

            $retryAfterMilliseconds = Get-GtRetryAfterMillisecond -ErrorRecord $errorRecord
            if ($null -ne $retryAfterMilliseconds) {
                if ($retryAfterMilliseconds -gt $MaxDelayMilliseconds) {
                    Write-PSFMessage -Level Warning -Message "$OperationName asked to wait $retryAfterMilliseconds ms (Retry-After), which exceeds the $MaxDelayMilliseconds ms limit; giving up" -Tag 'retry', 'ratelimit'
                    throw
                }
                $delayMilliseconds = $retryAfterMilliseconds
            }
            else {
                $exponentialDelay = [math]::Min($MaxDelayMilliseconds, $BaseDelayMilliseconds * [math]::Pow(2, $attempt - 1))
                $delayMilliseconds = Get-Random -Minimum 0 -Maximum ([int]$exponentialDelay + 1)
            }

            Write-PSFMessage -Level Warning -Message "$OperationName failed (attempt $attempt of $MaxAttempts): $($errorRecord.Exception.Message). Retrying in $delayMilliseconds ms" -Tag 'retry'
            Start-Sleep -Milliseconds $delayMilliseconds
        }
    }
}
```

## Using the Helper

Retry goes **inside** the lowest-level helper, and error reporting goes in the public function:

```powershell
# Private/Invoke-GtApiRequest.ps1
function Invoke-GtApiRequest {
    [CmdletBinding()]
    param (
        [ValidateSet('Get', 'Put', 'Patch', 'Delete', 'Post')]
        [string]$Method = 'Get',

        [Parameter(Mandatory)]
        [string]$Path,

        [hashtable]$Body,

        [hashtable]$Header,

        # POST is not idempotent: retry only when the caller confirms an idempotency key is sent
        [switch]$AllowPostRetry
    )

    $restParams = @{
        Uri         = (Get-PSFConfigValue -FullName 'GitHubTools.Api.BaseUri') + $Path
        Method      = $Method
        Headers     = $Header
        TimeoutSec  = Get-PSFConfigValue -FullName 'GitHubTools.Api.TimeoutSeconds'
        ErrorAction = 'Stop'
    }
    if ($PSBoundParameters.ContainsKey('Body')) {
        $restParams.Body = $Body | ConvertTo-Json -Depth 10
        $restParams.ContentType = 'application/json'
    }

    if ($Method -eq 'Post' -and -not $AllowPostRetry) {
        return Invoke-RestMethod @restParams
    }
    Invoke-GtWithRetry -OperationName "$Method $Path" -ScriptBlock { Invoke-RestMethod @restParams }
}
```

- `POST` is never retried unless the caller passes `-AllowPostRetry`, which it should do only when the request carries an idempotency key.
- The public function checks `ShouldProcess` **once**, before it calls the helper, and handles the final failure with `Stop-PSFFunction` (see `powershell-functions-rules.md`).

## Simple Fixed Retry with PSFramework

For local, non-HTTP operations where a fixed wait is enough:

```powershell
Invoke-PSFProtectedCommand -Action 'Writing report file' -Target $reportPath -ScriptBlock {
    Set-Content -Path $reportPath -Value $reportContent -Encoding UTF8 -ErrorAction Stop
} -RetryCount 3 -RetryWait 2s -RetryErrorType 'System.IO.IOException' -EnableException $EnableException -PSCmdlet $PSCmdlet -Continue
```

- Always pass `-RetryErrorType` (or `-RetryCondition`) so that only the transient exception types are retried.
- `Invoke-PSFProtectedCommand` also handles `ShouldProcess`. Don't add a second `ShouldProcess` check around it.

## Logging

- Log every retry at `Warning` level with the `retry` tag, including the operation name, attempt number, maximum attempts, delay, and error message.
- Log the final give-up once, at `Warning` level, before rethrowing. The public function's `Stop-PSFFunction` reports the failure to the user.
- Never log request bodies or headers that may contain tokens.

## Testing

- Mock `Start-Sleep` in every retry test, so tests run instantly and you can check the delays:

```powershell
BeforeAll {
    Mock -CommandName Start-Sleep -ModuleName GitHubTools
}

It 'retries a 503 and succeeds on the third attempt' {
    InModuleScope GitHubTools {
        $script:callCount = 0
        Mock -CommandName Test-GtTransientError -MockWith { $true }
        $result = Invoke-GtWithRetry -OperationName 'test' -MaxAttempts 5 -ScriptBlock {
            $script:callCount++
            if ($script:callCount -lt 3) { throw 'Service Unavailable' }
            'ok'
        }
        $result | Should-Be 'ok'
    }
    Should-Invoke Start-Sleep -ModuleName GitHubTools -Times 2 -Exactly
}
```

- Required cases:
  - succeeds on the first attempt (no sleep)
  - succeeds after transient failures
  - gives up after `MaxAttempts` and rethrows the **original** error
  - does **not** retry permanent errors (a single attempt)
  - honours `Retry-After`
  - gives up when `Retry-After` exceeds the maximum delay
  - every delay is ≤ `MaxDelayMilliseconds`
  - `POST` isn't retried unless `-AllowPostRetry` is passed
