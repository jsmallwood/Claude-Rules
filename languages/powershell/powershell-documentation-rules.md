---
paths:
  - "**/*.ps1"
  - "**/*.psm1"
  - "**/*.psd1"
  - "**/docs/**/*.md"
---

# PowerShell Documentation Rules — PlatyPS

[PlatyPS](https://github.com/PowerShell/platyPS) generates all `Get-Help` documentation for modules. The PlatyPS
markdown in `docs/` is the **source of truth**. The MAML XML that `Get-Help` reads is a build artifact generated from
that markdown.

## Which PlatyPS

- Modules are built with PowerShellBuild (see `powershell-modules-rules.md`), and PowerShellBuild requires the
  **`platyPS` module, version 0.14.1 or later** (0.14.x). Use that module and its cmdlets.
- Don't install or call **Microsoft.PowerShell.PlatyPS** (1.x) in a PowerShellBuild project. It's a different module
  with different cmdlet names and a different markdown schema, and mixing the two in one project or session breaks
  help generation. Revisit this only when PowerShellBuild itself moves to 1.x.
- Pin platyPS in `requirements.psd1` with the other build dependencies.

| Task | Command (platyPS 0.14.x) | PowerShellBuild task |
| --- | --- | --- |
| Create markdown for a module's commands | `New-MarkdownHelp -Module X -OutputFolder ./docs/en-US -WithModulePage` | `GenerateMarkdown` |
| Update markdown after code changes | `Update-MarkdownHelpModule -Path ./docs/en-US -RefreshModulePage` | `GenerateMarkdown` |
| Build MAML for `Get-Help` | `New-ExternalHelp -Path ./docs/en-US -OutputPath ./Output/X/<version>/en-US` | `GenerateMAML` / `BuildHelp` |

Import the module that contains the commands first (`Import-Module ./X/X.psd1 -Force`). PlatyPS reads parameter
metadata from the loaded commands. Prefer the PowerShellBuild tasks; run the task names exactly as
`./build.ps1 -Help` lists them.

## `.EXTERNALHELP` Is Mandatory on Public Functions

Comment-based help takes precedence over MAML. If a public function keeps a normal help block, `Get-Help` shows that
block and the built MAML is silently ignored. The `.EXTERNALHELP` keyword reverses the precedence.

```powershell
function Get-GtIssue {
    <#
    .EXTERNALHELP GitHubTools-help.xml
    .SYNOPSIS
        Gets an issue from a GitHub repository.
    #>
    [CmdletBinding()]
    param ()
}
```

- Put `.EXTERNALHELP` **inside the same `<# #>` block** as the one-line `.SYNOPSIS`, as the first entry, with the value on the same line as the keyword. Any other placement can be silently ignored, with no error or warning.
- The value is the **file name only**, no path. `Get-Help` looks in the module's culture folder (`en-US/`).
- Copy the file name **exactly** as the build produced it (check `Get-ChildItem ./Output/X/<version>/en-US`). platyPS 0.14 writes `<ModuleName>-help.xml`. The case matters on Linux and macOS: a mismatch is invisible on Windows and loses all help elsewhere.
- Add `.EXTERNALHELP` **after** the first markdown generation, not before. It suppresses comment-based help, and PlatyPS seeds new markdown from `Get-Help`, so adding it first leaves you with placeholders only.
- While developing, before MAML exists, `Get-Help` shows only syntax. That's expected. Build the help (`./build.ps1 -Task BuildHelp`) to see the real content.

## Workflow

1. **New public function:** write its help, generate its markdown with PlatyPS, then cut the function's comment block down to `.EXTERNALHELP` and a one-line `.SYNOPSIS`. Never write the markdown file from scratch.
2. **Changed parameters:** run the PlatyPS update so syntax and parameter blocks match the code. Don't edit those generated sections by hand. Check `[OutputType()]` and `[SupportsWildcards()]` yourself, because PlatyPS can only reflect what the attributes declare.
3. **Fill in content:** replace every placeholder (`{{ Fill in the Synopsis }}`, `{{ Fill in the Description }}` and so on) with real text. Generated help that still contains placeholders is not finished.
4. **Build:** generate the MAML into the built module's `en-US/` folder as part of the build, and ship it with the module.
5. **Verify:** `Import-Module` the **built** module, then run `Get-Help <Command> -Full` to confirm the help renders.

## Content Requirements (per command)

- **Synopsis:** one sentence of at most 80 characters that starts with a verb ("Gets an issue from a GitHub repository.").
- **Description:** what the command does, why you'd use it, and important behavior such as pipeline input, ShouldProcess, and side effects.
- **Examples:** at least **three**, progressing from basic use, to a realistic multi-parameter scenario, to pipeline use. Each example has a code block, a sentence explaining it, and the expected output or behavior. Use realistic values, never real secrets. Every example must actually run as written; cover it with a test (see `powershell-testing-rules.md`).
- **Parameters:** a description for every parameter, including its accepted values and default.
- **Inputs / Outputs:** the actual pipeline input types and `[OutputType()]` types.
- **Related links:** a bare command or `about_` topic name for anything inside the module, and an absolute `https://` URL for anything outside it. Never a relative file path: `Get-Help` throws on it at read time.
- Parameter names in help use the same PascalCase as the code (`-IssueTitle`).

## CI Gates

Fail the build when:

- any markdown under `docs/` still contains `{{ Fill in`
- a command is exported but has no markdown file, or a markdown file documents a command that's no longer exported
- a related link uses a relative path

```powershell
$unfilled = Select-String -Path ./docs/en-US/*.md -Pattern '\{\{\s*Fill in' -List
if ($unfilled) {
    throw "Help still contains placeholders: $(($unfilled.Path | Split-Path -Leaf) -join ', ')"
}
```

- Exclude generated help from markdown linting (`!**/docs/**/*.md` in the linter's glob list). PlatyPS output breaks rules such as MD013 and MD040, and every regeneration would bring the violations back.

## Comment-Based Help

- **Public module functions:** only `.EXTERNALHELP` and a one-line `.SYNOPSIS` (see above). Everything else lives in markdown.
- **Private functions:** full comment-based help (`.SYNOPSIS`, `.DESCRIPTION`, `.PARAMETER` for each parameter). PlatyPS never sees them, so the comment block is their only documentation.
- **Standalone scripts** (`.ps1` that aren't part of a module): full comment-based help (`.SYNOPSIS`, `.DESCRIPTION`, `.PARAMETER`, `.EXAMPLE`).
- Don't keep change history, version numbers, or dates in `.NOTES`. That belongs in `CHANGELOG.md` and git.

## About Topics

- Write module-level concepts and PSFramework configuration settings in `about_<ModuleName>.help.md`, and keep it up to date.
- List every configuration key the module declares with `Set-PSFConfig`, with its description, type, and default.
