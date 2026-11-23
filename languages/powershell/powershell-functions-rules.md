---
paths:
  - "**/*.ps1"
  - "**/*.psm1"
---

# PowerShell Function Rules

These rules apply to every function, public or private, in modules and in scripts. Naming and approved verbs are in `powershell-naming-rules.md`; style is
in `powershell-rules.md`. Error handling is in `powershell-error-handling-rules.md`.

## Naming & Files

- Name functions `<ApprovedVerb>-<Prefix><PascalCaseNoun>`: `Get-GtIssue`, `ConvertTo-GtIssueObject`. The verb must come from `Get-Verb`.
- Name parameters in PascalCase (`$IssueTitle`, `$RepoOwner`) and local variables in camelCase (`$apiHeader`, `$response`). Callers use the same PascalCase (`-RepoOwner`).
- In modules, put one function in each file and name the file after the function (`Public/Get-GtIssue.ps1`).
- Public functions are exported and documented with PlatyPS. Private functions use the same naming, are **never** exported, and live in `Private/`.

## Template

```powershell
function Set-GtIssue {
    <#
    .EXTERNALHELP GitHubTools-help.xml
    .SYNOPSIS
        Updates the title or state of a GitHub issue.
    #>
    [CmdletBinding(SupportsShouldProcess, ConfirmImpact = 'Medium', DefaultParameterSetName = 'ByNumber')]
    [OutputType('GitHubIssue')]
    param (
        [Parameter(Mandatory, ParameterSetName = 'ByNumber', ValueFromPipelineByPropertyName)]
        [ValidateRange(1, [int]::MaxValue)]
        [int]$IssueNumber,

        [Parameter(Mandatory, ParameterSetName = 'ByObject', ValueFromPipeline)]
        [PSTypeName('GitHubIssue')]
        [object]$InputObject,

        [Parameter(Mandatory)]
        [string]$RepoOwner,

        [Parameter(Mandatory)]
        [string]$RepoName,

        [ValidateLength(1, 256)]
        [string]$IssueTitle,

        [ValidateSet('open', 'closed')]
        [string]$State,

        [switch]$PassThru,

        [switch]$EnableException
    )

    begin {
        $apiHeader = Get-GtApiHeader
    }

    process {
        if ($PSCmdlet.ParameterSetName -eq 'ByObject') {
            $IssueNumber = $InputObject.number
        }

        $body = @{}
        if ($PSBoundParameters.ContainsKey('IssueTitle')) { $body.title = $IssueTitle }
        if ($PSBoundParameters.ContainsKey('State')) { $body.state = $State }

        if (-not $PSCmdlet.ShouldProcess("$RepoOwner/$RepoName#$IssueNumber", 'Update issue')) {
            return
        }

        try {
            $requestParams = @{
                Method = 'Patch'
                Path   = "/repos/$RepoOwner/$RepoName/issues/$IssueNumber"
                Body   = $body
                Header = $apiHeader
            }
            $response = Invoke-GtApiRequest @requestParams
            Write-PSFMessage -Level Verbose -Message "Updated issue $IssueNumber" -Target $IssueNumber -Tag 'issue'
            if ($PassThru) {
                ConvertTo-GtIssueObject -InputObject $response
            }
        }
        catch {
            $stopParams = @{
                Message         = "Failed to update issue $IssueNumber"
                ErrorRecord     = $_
                Target          = $IssueNumber
                EnableException = $EnableException
            }
            Stop-PSFFunction @stopParams -Continue
        }
    }
}
```

## CmdletBinding & Output

- Every function is an advanced function: `[CmdletBinding()]` and a `param ()` block, even when there are no parameters.
- Declare `[OutputType()]` for every type the function emits. Emit **one** type per parameter set.
- Write objects to the output stream. Don't use `return $collection`. Use `return` only to exit early.
- Never call `Format-*`, `Out-Host`, or `Write-Host` to produce data. Formatting belongs to the caller or to `.format.ps1xml` files.
- Return rich objects (classes or `[PSCustomObject]` with `PSTypeName`), never preformatted strings.
- Keep unintended output out of the stream: assign it to `$null` (`$null = $list.Add($item)`). Prefer `$null =` over `| Out-Null` because it's faster.

## Parameters

- Give each parameter a type. Use `[object]` only when the function really accepts anything.
- Validate input with attributes instead of manual checks in the body: `[ValidateNotNullOrEmpty()]`, `[ValidateSet()]`, `[ValidateRange()]`, `[ValidatePattern()]`, `[ValidateLength()]`, `[ValidateScript()]`. `[ValidateScript()]` must throw a clear message.
- Don't add `[ValidateNotNullOrEmpty()]` to a `Mandatory` `[string]` parameter. Mandatory already rejects `$null` and `''`, so the attribute is redundant. Use it on **optional** parameters. Mandatory does **not** reject whitespace-only strings; when that matters, add `[ValidatePattern('\S')]`.
- **Never use a module class as a type in a `param ()` block or in `[OutputType()]`** (`[GitHubIssue]$InputObject`, `[OutputType([GitHubIssue])]`). Type literals in attributes are resolved when the file is parsed, and a dot-sourced file can't see a class defined in a sibling file at that moment. One such parameter breaks the import of the whole module. Use `[PSTypeName('GitHubIssue')] [object]$InputObject` and `[OutputType('GitHubIssue')]` (string form). Casts and `::new()` inside the function body are fine, because they resolve at run time.
- Use `[switch]` for flags, never `[bool]`. A switch never defaults to `$true`.
- Use parameter sets for mutually exclusive inputs, and always set `DefaultParameterSetName`.
- Make a parameter positional only when it's the obvious primary input (usually `Position = 0`). Everything else is named.
- Add `[Alias()]` only for well-known alternative names (for example `[Alias('Id')]`), and never use one in place of a properly named parameter.
- Credentials: `[PSCredential] [System.Management.Automation.Credential()] $Credential = [PSCredential]::Empty`. Never accept a plain-text password `[string]`.
- To tell whether a parameter was supplied, check `$PSBoundParameters.ContainsKey('name')`. Don't compare against `$null` or a default.
- Avoid dynamic parameters. Use `[ArgumentCompleter()]` or `Register-ArgumentCompleter` for tab completion.
- Every public function has an `-EnableException` switch (see `powershell-logging-rules.md`).

## Pipeline

- Accept pipeline input where it's natural (`ValueFromPipeline` for the main object, `ValueFromPipelineByPropertyName` for identifiers).
- `begin {}` runs one-time setup, `process {}` handles each item, and `end {}` does aggregation and cleanup. If you use `process`, put **all** per-item code in it.
- Stream results from `process {}` as they're produced. Don't collect them and emit at the end unless you need to sort or aggregate.
- **`return` in `begin {}` doesn't stop the command.** It ends the `begin` block only, and `process {}` still runs for every item. To stop, record the decision in a variable (or call `Stop-PSFFunction`) and check it at the top of `process {}` (`if (Test-PSFFunctionInterrupt) { return }`).
- **Empty collections unroll to `$null`.** A function that outputs `@()` gives its caller `$null`, and a one-element array arrives as the bare element. Wrap the call site in `@(...)` when you need an array (`$issues = @(Get-GtIssue ...)`), or emit with `Write-Output -NoEnumerate` when the function must return the collection as one object.
- **Piped and parameter input run different code.** Passed as `-InputObject $collection`, the whole collection reaches `process {}` once; piped, `process {}` runs once per element. A function that reduces its input to one result (a sum, a merged outcome) must collect items in `process {}` and reduce in `end {}`, or it returns one wrong result per element. Exclude `[string]` and `[hashtable]` before any `IEnumerable` check, because both enumerate. Test both paths (see `powershell-testing-rules.md`).
- In a command that supports `-WhatIf`, don't enumerate with `ForEach-Object -MemberName`. That parameter set honours `ShouldProcess`, so under `-WhatIf` it returns nothing. Use a script block (`ForEach-Object { $_.Name }`).

## ShouldProcess

- Any function that changes state (`New`, `Set`, `Remove`, `Add`, `Update`, `Start`, `Stop`, and `Invoke` with side effects) declares `SupportsShouldProcess`.
- Call `$PSCmdlet.ShouldProcess($target, $action)` immediately before the change, and only once per item.
- Use `ConfirmImpact = 'High'` for destructive or irreversible operations. Add a `-Force` switch that bypasses confirmation: `if ($Force -or $PSCmdlet.ShouldProcess(...))`.
- When a function calls other state-changing functions, `-WhatIf` and `-Confirm` pass to them automatically. Don't pass them explicitly.
- Wrap the ShouldProcess check **around** any retry logic, never inside it, so the user is asked only once.

## Scope & State

- Functions don't read or write `$global:` variables.
- Module state lives in `$script:` scope, and is only accessed through private getter/setter functions.
- Settings come from PSFramework Configuration (`Get-PSFConfigValue`), not from variables (see `powershell-configuration-rules.md`).
- Don't change preference variables (`$ErrorActionPreference`, `$VerbosePreference`) inside functions. Use `-ErrorAction` / `-Verbose` on individual calls.
- **Background work can't see module state.** Code inside `Start-Job`, `Start-ThreadJob`, a runspace, or `ForEach-Object -Parallel` runs without this session's `$script:` variables, and jobs and runspaces don't have the module's private functions either. Nothing fails loudly; values are just `$null`. Pass everything in through `-ArgumentList`, `$using:`, or parameters, and import the module inside the block if it needs module functions. Pester mocks don't reach these workers either.

## Data From APIs and Files

The same JSON can arrive as a hashtable (`ConvertFrom-Json -AsHashtable`, Microsoft Graph SDK by default) or as a
`PSCustomObject` (`ConvertFrom-Json`). Index syntax (`$response['value']`) silently returns `$null` on a
`PSCustomObject`, and member syntax returns a hashtable's own `Count`/`Keys` when the key is missing. Read response
data through one private accessor that handles both shapes:

```powershell
function Get-GtResponseProperty {
    [CmdletBinding()]
    [OutputType([object])]
    param (
        [Parameter(Mandatory)]
        [AllowNull()]
        [object]$InputObject,

        [Parameter(Mandatory)]
        [string]$Name
    )

    if ($null -eq $InputObject) { return $null }
    if ($InputObject -is [System.Collections.IDictionary]) {
        if ($InputObject.Contains($Name)) { return $InputObject[$Name] }
        return $null
    }
    $property = $InputObject.PSObject.Properties[$Name]
    if ($null -ne $property) { return $property.Value }
    return $null
}
```

- Wrap reads of collections in `@(...)` (`@(Get-GtResponseProperty -InputObject $response -Name 'value')`), because the accessor unrolls them like any function.
- Test public functions with both shapes of the same recorded fixture.

## Size & Design

- Each function does one thing. If its description needs the word "and", split it.
- Aim for functions under about 50 lines of logic, and move helpers into private functions.
- Public functions validate input, call private helpers, and shape output. HTTP, file, and parsing details go in private functions.
- Functions in modules don't call `exit`, prompt the user with `Read-Host`, or change the current location without restoring it.

## Documentation & Tests

- Public functions: write PlatyPS help, and keep only `.EXTERNALHELP` and a one-line `.SYNOPSIS` in the function (see `powershell-documentation-rules.md`).
- Every function, public or private, has Pester tests covering the success path, failure path, parameter validation, pipeline input, and `-WhatIf` (see `powershell-testing-rules.md`).
