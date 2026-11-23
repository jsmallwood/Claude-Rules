---
paths:
  - "**/*.psm1"
  - "**/*.psd1"
  - "**/psakeFile.ps1"
  - "**/build.ps1"
---

# PowerShell Module Rules — Structure & PowerShellBuild

All modules use a standard directory structure and are built, tested, documented, and published with
[PowerShellBuild](https://github.com/psake/PowerShellBuild) running under [psake](https://github.com/psake/psake).
Function and class rules are in `powershell-functions-rules.md` and `powershell-classes-rules.md`.

## Directory Structure

```text
GitHubTools/                          # Repository root
├── .github/workflows/ci.yml          # CI: ./build.ps1 -Task Test on every target version/OS
├── build.ps1                         # Entry point: bootstrap dependencies, invoke psake
├── psakeFile.ps1                     # Build configuration ($PSBPreference) and task list
├── requirements.psd1                 # PSDepend: pinned build/test dependency versions
├── PSScriptAnalyzerSettings.psd1     # Static analysis settings (mandatory)
├── README.md
├── CHANGELOG.md
├── LICENSE
├── GitHubTools/                      # Module source (folder name = module name)
│   ├── GitHubTools.psd1              # Source manifest — explicit exports
│   ├── GitHubTools.psm1              # Dev loader: dot-sources the folders below in order
│   ├── Enum/                         # One enum per file
│   ├── Classes/                      # One class per file (base classes first)
│   ├── Configurations/               # Set-PSFConfig -Initialize declarations
│   ├── Private/                      # Non-exported functions, one per file
│   ├── Public/                       # Exported functions, one per file
│   └── en-US/
│       └── about_GitHubTools.help.md # About topic source
├── docs/
│   └── en-US/                        # PlatyPS markdown (source of truth for Get-Help)
├── tests/
│   ├── Module.Tests.ps1              # Manifest, exports, help, analyzer meta-tests
│   ├── Public/                       # Get-GtIssue.Tests.ps1 …
│   ├── Private/
│   └── Classes/
└── Output/                           # Build output — git-ignored, never edited by hand
    └── GitHubTools/<version>/        # Built module (compiled .psm1, manifest, MAML help)
```

- `Output/` and test result files are build artifacts. Add them to `.gitignore`.
- Never edit files in `Output/`. Change the source and rebuild.
- PowerShellBuild finds the project paths through BuildHelpers (`Set-BuildEnvironment`). After you change the layout, run `./build.ps1 -Task Init` and check the paths it reports. Don't assume them.

## Manifest (`.psd1`)

```powershell
@{
    RootModule           = 'GitHubTools.psm1'
    ModuleVersion        = '1.2.0'
    GUID                 = '<generate once with New-Guid, never change>'
    Author               = '<author>'
    Description          = 'Manage GitHub issues from PowerShell.'
    PowerShellVersion    = '5.1'
    CompatiblePSEditions = @('Desktop', 'Core')
    RequiredModules      = @('PSFramework')
    FunctionsToExport    = @('Get-GtIssue', 'New-GtIssue', 'Set-GtIssue', 'Remove-GtIssue', 'Invoke-GtIssueAction')
    CmdletsToExport      = @()
    VariablesToExport    = @()
    AliasesToExport      = @()
    PrivateData          = @{
        PSData = @{
            Tags         = @('GitHub', 'Issues')
            ProjectUri   = '<repository url>'
            LicenseUri   = '<license url>'
            ReleaseNotes = 'See CHANGELOG.md'
        }
    }
}
```

- **Never** use `'*'` or omit an export list. Wildcard or missing exports force PowerShell to load the whole module to discover commands, which slows auto-loading ([module authoring considerations](https://learn.microsoft.com/en-us/powershell/scripting/dev-cross-plat/performance/module-authoring-considerations?view=powershell-7.6)). `FunctionsToExport` must match the file names in `Public/` exactly, and a meta-test in `tests/Module.Tests.ps1` enforces this.
- Use SemVer for `ModuleVersion`: a breaking change bumps major, a new feature bumps minor, a fix bumps patch. Update `CHANGELOG.md` in the same change.
- Pin `RequiredModules` with `@{ ModuleName = 'PSFramework'; ModuleVersion = '<min>' }` when you depend on a minimum version.
- `Test-ModuleManifest` must pass.

## Source Loader (`.psm1`)

When the module isn't compiled, the source `.psm1` loads files in dependency order:

```powershell
$sourceFolders = @('Enum', 'Classes', 'Configurations', 'Private', 'Public')
foreach ($folder in $sourceFolders) {
    $folderPath = Join-Path -Path $PSScriptRoot -ChildPath $folder
    if (-not (Test-Path -Path $folderPath)) { continue }
    foreach ($file in Get-ChildItem -Path $folderPath -Filter '*.ps1' -Recurse | Sort-Object -Property FullName) {
        . $file.FullName
    }
}
```

- Don't call `Export-ModuleMember` with wildcards. The manifest controls what's exported.
- Module-level initialization (TLS settings on 5.1, type accelerators) stays small and goes in its own file in the load list, so it ends up in the compiled module too.
- Don't run any code at import time that does network calls, prompts, or slow work.

## PowerShellBuild

### Dependencies (`requirements.psd1`)

```powershell
@{
    PSDependOptions   = @{ Target = 'CurrentUser' }
    'psake'           = '4.9.0'
    'PowerShellBuild' = '0.6.2'
    'BuildHelpers'    = '2.0.16'
    'Pester'          = @{ Version = '6.2.0'; Parameters = @{ SkipPublisherCheck = $true } }
    'PSScriptAnalyzer'= '1.25.0'
    'platyPS'         = '0.14.2'
    'PSFramework'     = '1.12.346'
}
```

- Pin exact versions. Upgrade them on purpose in a separate change. The versions above are examples; use the ones the project has validated.
- PowerShellBuild requires **platyPS 0.14.1 or later** (0.14.x) for its help tasks. Pin it here, and don't install `Microsoft.PowerShell.PlatyPS` (1.x) in the same project (see `powershell-documentation-rules.md`).
- Check that the pinned PowerShellBuild version's `Pester` task supports Pester 6 before upgrading Pester; if it doesn't, keep both pins in step.

### Entry Point (`build.ps1`)

```powershell
[CmdletBinding(DefaultParameterSetName = 'Task')]
param (
    [Parameter(ParameterSetName = 'Task', Position = 0)]
    [string[]]$Task = 'Default',

    [Parameter(ParameterSetName = 'Task')]
    [switch]$Bootstrap,

    [Parameter(ParameterSetName = 'Help')]
    [switch]$Help
)

$ErrorActionPreference = 'Stop'

if ($Bootstrap) {
    Install-Module -Name PSDepend -Repository PSGallery -Scope CurrentUser -Force
    Import-Module -Name PSDepend
    Invoke-PSDepend -Path (Join-Path -Path $PSScriptRoot -ChildPath 'requirements.psd1') -Install -Import -Force
}

$psakeFile = Join-Path -Path $PSScriptRoot -ChildPath 'psakeFile.ps1'
if ($PSCmdlet.ParameterSetName -eq 'Help') {
    Get-PSakeScriptTasks -buildFile $psakeFile | Format-Table -Property Name, Description, Alias, DependsOn
}
else {
    Set-BuildEnvironment -Force
    Invoke-psake -buildFile $psakeFile -taskList $Task -nologo
    exit ([int](-not $psake.build_success))
}
```

### Build Configuration (`psakeFile.ps1`)

```powershell
properties {
    $PSBPreference.Build.CompileModule                 = $true
    $PSBPreference.Build.CompileDirectories            = @('Enum', 'Classes', 'Configurations', 'Private', 'Public')
    $PSBPreference.Help.DefaultLocale                  = 'en-US'
    $PSBPreference.Test.OutputFile                     = 'Output/testResults.xml'
    $PSBPreference.Test.ScriptAnalysis.Enabled         = $true
    $PSBPreference.Test.ScriptAnalysis.SettingsPath    = Join-Path -Path $PSScriptRoot -ChildPath 'PSScriptAnalyzerSettings.psd1'
    $PSBPreference.Test.ScriptAnalysis.FailBuildOnSeverityLevel = 'Warning'
    $PSBPreference.Test.CodeCoverage.Enabled           = $true
    $PSBPreference.Test.CodeCoverage.Threshold         = 0.80
}

task Default -depends Test

task Test -FromModule PowerShellBuild -minimumVersion '0.6.2'

task Publish -FromModule PowerShellBuild -minimumVersion '0.6.2'
```

- Configure the build only through `$PSBPreference` in `properties {}`. Don't copy PowerShellBuild's task code into the psakeFile.
- `CompileModule = $true` produces a single `.psm1` in `Output/`. Always ship the compiled module: dot-sourcing many files at import is noticeably slower. `CompileDirectories` **must** follow the load order Enum → Classes → Configurations → Private → Public.
- After the first compiled build, and whenever the layout changes, open the compiled `.psm1` in `Output/` and confirm that every folder is included in the right order.
- `FailBuildOnSeverityLevel = 'Warning'` matches the PSScriptAnalyzer rule in `powershell-rules.md` (no warnings allowed).
- Tests run against the **built** module in `Output/`, not the source folder.

### Standard Commands

| Goal | Command |
| --- | --- |
| First run / new machine | `./build.ps1 -Bootstrap` |
| List available tasks | `./build.ps1 -Help` |
| Clean, build, analyze, test (default) | `./build.ps1` or `./build.ps1 -Task Test` |
| Build only | `./build.ps1 -Task Build` |
| Static analysis only | `./build.ps1 -Task Analyze` |
| Pester only | `./build.ps1 -Task Pester` |
| Regenerate PlatyPS markdown / MAML | `./build.ps1 -Task GenerateMarkdown` / `./build.ps1 -Task BuildHelp` |
| Publish | `./build.ps1 -Task Publish` |

- Before you call a change finished, run `./build.ps1 -Task Test` and report the real result. If you couldn't run it, say so.
- Run the task names exactly as `./build.ps1 -Help` lists them for the installed PowerShellBuild version. If a task isn't listed, don't invent it.

### Publishing

- Publish only from CI, from a tagged commit whose tag matches `ModuleVersion`. The release job reruns every gate and refuses a tag that disagrees with the manifest.
- Supply the API key with `$PSBPreference.Publish.PSRepositoryApiKey = $env:PSGALLERY_API_KEY` (a CI secret). Never commit keys or put them in `psakeFile.ps1`.
- A Gallery version can be unlisted but **never deleted**, so get it right before upload:
  - Publish only the built module in `Output/`, never the repository root (that would ship `.git`, tests, and build scripts). Before publishing, import the built copy in a fresh process and check its exports.
  - The Gallery rejects metadata that passes every local check. Measure these in a meta-test (`tests/Module.Tests.ps1`) so a pull request fails, not the release:

    | Field | Limit |
    | --- | --- |
    | `Description` | 4,000 characters |
    | Full tag list: your `Tags` plus the generated `PSModule`, `PSEdition_<edition>`, `PSIncludes_Function`, and a `PSFunction_<name>` and `PSCommand_<name>` per exported function | 4,000 characters joined with spaces |
    | `ReleaseNotes` | 10,600 characters |

  - Keep only the current release's notes in `ReleaseNotes`, ending with a pointer to `CHANGELOG.md`. Each exported command adds roughly twice its name length to the tag list.
  - Leave a URI key (`IconUri`, `HelpInfoURI`, `ProjectUri`, `LicenseUri`) out of the manifest until it has a value. An empty string (`IconUri = ''`) makes packing fail with an error that names neither the manifest nor the key.
- **`-WhatIf` reaches everything a release script calls.** It sets `$WhatIfPreference`, which child scopes inherit, so a `SupportsShouldProcess` release script run with `-WhatIf` also skips its own staging, copying, and help-building steps. The rehearsal then verifies nothing. Gate only the publish step with `ShouldProcess`, pin every other step with `-WhatIf:$false`, and test `-WhatIf` together with the script's other switches from a clean checkout.

## CI

- Run `./build.ps1 -Bootstrap -Task Test` on every pull request.
- Test on every targeted edition and OS: `powershell` (5.1, Windows) when `Desktop` is supported, and `pwsh` on Windows/Linux/macOS when `Core` is supported.
- Publish test results (`Output/testResults.xml`) and fail the pipeline when analysis, tests, or coverage fail.
