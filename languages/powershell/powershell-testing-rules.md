---
paths:
  - "**/*.Tests.ps1"
  - "**/tests/**/*.ps1"
  - "**/PesterConfiguration.psd1"
  - "**/Pester.BeforeContainer.ps1"
---

# PowerShell Testing Rules — Pester

[Pester](https://pester.dev) is the **only** framework allowed for testing and mocking PowerShell code. Target
**Pester 6.2+**, which runs on Windows PowerShell 5.1 and PowerShell 7.4+. Pester 3/4 patterns are not allowed.

## Layout & Naming

- Put tests in `tests/`, mirroring the source tree: `tests/Public/Get-GtIssue.Tests.ps1`.
- Name each test file after the code it tests, with the `.Tests.ps1` suffix.
- Write `Describe` for the function, `Context` for the scenario, and `It` for the expected behavior in plain language.
  - `It 'returns an issue object when the issue exists'`
- **Tag every `Describe`** (`-Tag 'Unit'`, `'Integration'`, `'Public'`, `'Private'`) so runs can be filtered. Never use the tag `None`.
- The naming conventions from `powershell-naming-rules.md` apply in tests too: camelCase variables, approved verbs for helper functions.
- Use plain ASCII in test names and output. No emoji.

## Structure

```powershell
#Requires -Modules @{ ModuleName = 'Pester'; ModuleVersion = '6.2.0' }

BeforeAll {
    # Test the BUILT module (see powershell-modules-rules.md). PowerShellBuild/BuildHelpers set $env:BHBuildOutput.
    if (-not (Get-Module -Name GitHubTools)) {
        Import-Module -Name (Join-Path -Path $env:BHBuildOutput -ChildPath 'GitHubTools.psd1')
    }
}

Describe 'Get-GtIssue' -Tag 'Unit', 'Public' {
    Context 'when the issue exists' {
        BeforeAll {
            Mock -CommandName Invoke-RestMethod -ModuleName GitHubTools -MockWith {
                [PSCustomObject]@{ number = 42; title = 'Bug'; state = 'open' }
            }
        }

        It 'returns the issue number' {
            $result = Get-GtIssue -RepoOwner 'octo' -RepoName 'demo' -IssueNumber 42
            $result.number | Should-Be 42
        }

        It 'calls the API exactly once' {
            Get-GtIssue -RepoOwner 'octo' -RepoName 'demo' -IssueNumber 42
            Should-Invoke Invoke-RestMethod -ModuleName GitHubTools -Times 1 -Exactly
        }
    }

    Context 'when the API fails' {
        BeforeAll {
            Mock -CommandName Invoke-RestMethod -ModuleName GitHubTools -MockWith { throw 'Not Found' }
        }

        It 'throws when EnableException is set' {
            { Get-GtIssue -RepoOwner 'octo' -RepoName 'demo' -IssueNumber 1 -EnableException } | Should-Throw
        }
    }
}
```

## Every Test File Is Self-Contained

Pester 6 discovers and runs **one file at a time**, so setup done by another file doesn't carry over.

- Each file imports the modules it needs: in `BeforeAll` for run time, and also in `BeforeDiscovery` if `-ForEach` data or the test tree depends on them.
- Import only when the module **isn't already loaded** (`if (-not (Get-Module -Name X))`). Don't `Import-Module -Force` or `Remove-Module` in every file; it's slow and adds nothing. Keep exactly one forced import, in `tests/Module.Tests.ps1`, which checks the freshly imported module.
- Because the module is shared across files, after you edit the module rebuild it and run the suite in a fresh process (`./build.ps1 -Task Test` does this).
- Shared bootstrap goes in `Pester.BeforeContainer.ps1`, not in "a file that runs first". Its top-level code runs at discovery only, so put run-time setup in its `BeforeAll`.

## Assertions (Pester 6)

Use the `Should-*` assertions in new tests. Existing `Should -Be` style still works, but don't mix styles within one file.

| Pester 5 | Pester 6 |
| --- | --- |
| `Should -Be $x` | `Should-Be $x` |
| `Should -BeOfType [T]` | `Should-HaveType ([T])` |
| `Should -BeTrue` / `-BeFalse` | `Should-BeTrue` / `Should-BeFalse` (strict `[bool]`) |
| `Should -BeNullOrEmpty` | `Should-BeNull`, `Should-BeEmptyString`, or `Should-BeFalsy`: pick the one you mean |
| `Should -HaveCount 3` | `Should-BeCollection -Count 3` |
| `Should -Match 'regex'` | `Should-MatchString 'regex'` |
| `Should -Throw '*msg*'` | `Should-Throw -ExceptionMessage 'msg'` |
| `Should -Invoke Cmd` | `Should-Invoke Cmd` |
| `Should -Not -Invoke Cmd` | `Should-NotInvoke Cmd` |
| `Should -InvokeVerifiable` | `Should-Invoke -Verifiable` |

## Rules

- Put all setup code in `BeforeAll` / `BeforeEach`. Loose code inside `Describe` or `Context` runs during discovery, not during the run.
- Use `-ForEach` (or `-TestCases`) for data-driven tests instead of copying the same `It` block. The data must never be empty: in Pester 6, `-ForEach` over `$null` or `@()` fails discovery. Never use `Input` as a test-data key; it collides with the automatic `$input` variable.
- Mock dependencies the module calls with `Mock -ModuleName <ModuleName>`.
- Test private functions with `InModuleScope <ModuleName> { ... }`. Use it sparingly, and prefer testing behavior through public functions. Never export a function just to test it.
- Check mocks with `Should-Invoke`, and use `-ParameterFilter { }` to check the arguments that were passed.
- Mocks don't reach code running in jobs, thread jobs, runspaces, or `ForEach-Object -Parallel`. Test the worker logic directly instead.
- Use `TestDrive:` for file system tests and `TestRegistry:` for registry tests (Windows only). Never touch real paths.
- Mock every external side effect: REST calls, file writes outside `TestDrive:`, registry, network, `Start-Process`.
- Mock `Write-PSFMessage` only when you need to assert that something was logged. Leave it unmocked otherwise.
- To test PSFramework configuration, set values with `Set-PSFConfig` in `BeforeAll`. Don't edit config files.
- Each `It` block checks one behavior. Every test must be able to run on its own and in any order.
- For functions that accept pipeline input, test **both** `$collection | Verb-Noun` and `Verb-Noun -InputObject $collection`, piping the shape the real upstream command produces. These run different code (see `powershell-functions-rules.md`).
- For functions that read API or file data, test with both the hashtable and the `PSCustomObject` shape of the same recorded fixture.
- Every example in a command's help or the README has a test that runs it as written.
- Never use removed syntax: `Should Be` (without the dash), `Assert-MockCalled`, `Assert-VerifiableMock`, `-Focus`, `Set-ItResult -Pending`, or `Invoke-Pester -Script`.
- Don't enable the experimental `Run.Parallel` or `Mock.Global` options unless asked.

## Running Tests

Configure runs with `New-PesterConfiguration`:

```powershell
$config = New-PesterConfiguration
$config.Run.Path = './tests'
$config.Run.PassThru = $true
$config.Output.Verbosity = 'Detailed'
$config.TestResult.Enabled = $true
$config.TestResult.OutputFormat = 'NUnitXml'
$config.CodeCoverage.Enabled = $true
$config.CodeCoverage.Path = './GitHubTools/Public', './GitHubTools/Private'
$config.CodeCoverage.CoveragePercentTarget = 80
$result = Invoke-Pester -Configuration $config

# A file that fails discovery adds nothing to FailedCount, so check containers too.
if ($result.FailedCount -gt 0 -or $result.FailedContainersCount -gt 0) {
    throw "Tests failed: $($result.FailedCount) test(s), $($result.FailedContainersCount) container(s)"
}
# CoveragePercentTarget only reports; enforce the gate yourself.
if ($result.CodeCoverage.CoveragePercent -lt 80) {
    throw "Code coverage $([math]::Round($result.CodeCoverage.CoveragePercent, 2))% is below 80%"
}
```

- In CI, turn on `Run.Shuffle` so tests that depend on declaration order are caught. The run prints the seed; replay it with `Run.ShuffleSeed`. Opt a file out with `#pester:no-shuffle` only with a comment explaining why.
- When filtering by tag, read the **passed** count, not the discovered count.
- Every public function needs tests for its success path, its failure path, and parameter validation.
- Aim for at least 80% code coverage on `Public/` and `Private/`.
- If a module targets both editions, run the tests on **both** `powershell.exe` (5.1) and `pwsh` (7.x).
- Report test results as they are. If you didn't run the tests, say so, and never describe tests you didn't run as passing.

## Recommended Meta-Tests

Keep a `tests/Module.Tests.ps1` that checks:

- `Test-ModuleManifest` passes.
- Every exported function uses an approved verb (`(Get-Verb).Verb -contains $verb`).
- Every exported function has PlatyPS-generated help (`Get-Help` returns a synopsis and at least three examples).
- `Invoke-ScriptAnalyzer` returns no findings for the module source.
- The manifest's `Description`, full tag list, and `ReleaseNotes` are within PowerShell Gallery limits (see `powershell-modules-rules.md`).
