---
paths:
  - "**/*.ps1"
  - "**/*.psm1"
---

# PowerShell Class & Enum Rules

These rules apply to every `class` and `enum` definition. Naming comes from `powershell-naming-rules.md` and style from `powershell-rules.md`.

## When to Use Classes

- Use classes for **data models** (`GitHubIssue`), **typed options/filters** (`IssueFilter`), and internal helpers that benefit from encapsulation.
- The **public API of a module is functions**, not classes. Users call `Get-GtIssue`, not `[GitHubIssue]::Load()`.
- Don't build a class hierarchy when a `[PSCustomObject]` with a `PSTypeName` would do.
- Use enums instead of magic strings for a fixed set of values.

## Naming

| Element | Convention | Example |
| --- | --- | --- |
| Class / enum type | PascalCase | `class GitHubIssue`, `enum IssueState` |
| Properties | camelCase (the same as variables) | `[string]$issueTitle` |
| Methods | PascalCase (the .NET convention) | `[string] ToMarkdown()` |
| Method parameters | camelCase (the .NET convention; function and script parameters are PascalCase) | `[void] AddLabel([string]$labelName)` |
| Enum values | camelCase | `open`, `closed`, `inProgress` |
| Static constants | camelCase, `static hidden` | `static hidden [int] $maxTitleLength = 256` |

## Files & Load Order

- Put one class or enum in each file, and name the file after the type: `Classes/GitHubIssue.ps1`, `Enum/IssueState.ps1`.
- Enums go in `Enum/` and classes go in `Classes/`.
- Load order matters because types are resolved when the script is parsed: **Enum → Classes (base classes before derived) → Private → Public**. Keep this order both in the source `.psm1` and in PowerShellBuild's `CompileDirectories` (see `powershell-modules-rules.md`).
- When classes depend on each other, control their order with numeric file prefixes, or keep the base class in the same file as the class that derives from it. Never rely on alphabetical order working out by chance.
- Types used in a class's signatures must already be loaded when the class is parsed. Declare them with `using namespace` / `using assembly` at the top of the file, or with `RequiredAssemblies` in the manifest. A class can't use a type from a module that's imported at runtime.

## Definition Template

```powershell
class GitHubIssue {
    # Properties — always typed
    [ValidateRange(1, [int]::MaxValue)]
    [int]$number

    [ValidateNotNullOrEmpty()]
    [string]$issueTitle

    [IssueState]$state = [IssueState]::open

    [System.Collections.Generic.List[string]]$labels = [System.Collections.Generic.List[string]]::new()

    hidden [datetime]$retrievedAt

    static hidden [int] $maxTitleLength = 256

    # Constructors
    GitHubIssue() {
        $this.Init(@{})
    }

    GitHubIssue([hashtable]$properties) {
        $this.Init($properties)
    }

    hidden [void] Init([hashtable]$properties) {
        foreach ($key in $properties.Keys) {
            $this.$key = $properties[$key]
        }
        $this.retrievedAt = [datetime]::UtcNow
    }

    # Methods
    [void] AddLabel([string]$labelName) {
        if ([string]::IsNullOrWhiteSpace($labelName)) {
            throw [System.ArgumentException]::new('Label name cannot be empty.', 'labelName')
        }
        if (-not $this.labels.Contains($labelName)) {
            $this.labels.Add($labelName)
        }
    }

    [string] ToString() {
        return "#$($this.number) $($this.issueTitle)"
    }
}
```

## Rules

- **Type every property and method parameter.** Untyped members are not allowed.
- Always refer to members with `$this.`. A bare `$number` inside a method is a local variable, not the property.
- Methods with a return type must `return` explicitly. Pipeline output inside a method is discarded, so nothing is "output" except the return value.
- `[void]` methods don't return anything.
- Validation attributes on properties (`[ValidateNotNullOrEmpty()]`, `[ValidateRange()]`, `[ValidateSet()]`) are encouraged. They're enforced whenever the property is assigned.
- PowerShell constructors can't chain (`: this()`). Put shared construction logic in a `hidden [void] Init()` method.
- Always define a parameterless constructor alongside parameterised ones, so the type can be created with `[GitHubIssue]@{ ... }` and deserialised.
- Mark internal members as `hidden`. `hidden` hides them from `Get-Member` and tab completion but **doesn't** enforce privacy, so don't rely on it for security.
- Override `ToString()` on data models so they display usefully in logs and strings.
- Implement .NET interfaces (`[System.IComparable]`, `[System.IEquatable[GitHubIssue]]`) when sorting or equality matters. Don't invent `Compare()` methods.
- Keep inheritance shallow (at most 2 levels). Prefer composition.
- Use static methods only for factories and pure helpers (`static [GitHubIssue] FromApiResponse([object]$response)`), never for shared mutable state.

## Errors & Logging in Classes

- Methods **throw .NET exceptions** (`[System.ArgumentException]`, `[System.InvalidOperationException]`, and so on). `Stop-PSFFunction` and `-EnableException` are for functions only. The calling function catches the exception and handles it as described in `powershell-error-handling-rules.md`.
- Methods may log with `Write-PSFMessage -Level Verbose` / `Debug`, and must never call `Write-Host`.
- Constructors validate their inputs and throw on invalid state, so an object is never left half-built.

## Exporting Classes from a Module

Classes in a module are visible to callers only through `using module <ModuleName>`, not `Import-Module`.

- Default: keep classes internal, and expose them through functions (`New-GtIssueFilter` returns an `[IssueFilter]`).
- If callers must reference the types directly (`[GitHubIssue]`), register them as type accelerators in the `.psm1`, as described in Microsoft's `about_Classes_and_Modules`, and remove them in `OnRemove`:

```powershell
$exportableTypes = @([GitHubIssue], [IssueFilter])
$typeAccelerators = [psobject].Assembly.GetType('System.Management.Automation.TypeAccelerators')
$existingAccelerators = $typeAccelerators::Get
foreach ($type in $exportableTypes) {
    if ($type.FullName -in $existingAccelerators.Keys) {
        throw [System.InvalidOperationException]::new("Type accelerator '$($type.FullName)' already exists.")
    }
}
foreach ($type in $exportableTypes) {
    $null = $typeAccelerators::Add($type.FullName, $type)
}
$MyInvocation.MyCommand.ScriptBlock.Module.OnRemove = {
    foreach ($type in $exportableTypes) {
        $null = $typeAccelerators::Remove($type.FullName)
    }
}.GetNewClosure()
```

## Development & Testing

- A class can't be redefined in a running session. After you change a class, start a new session (or `pwsh -NoProfile`). Run Pester in a fresh process (`pwsh -NoProfile -Command Invoke-Pester ...`) so tests don't use stale types.
- In tests, get at the types with `using module <path to built .psd1>` at the top of the test file, or with `InModuleScope`.
- Test constructors (valid and invalid input), property validation, every public method, and `ToString()`.
- Windows PowerShell 5.1 and PowerShell 7 class support differs in places. Check `powershell-5.1-rules.md` / `powershell-7-rules.md`, and run tests on every targeted version.
