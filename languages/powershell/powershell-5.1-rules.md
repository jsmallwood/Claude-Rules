---
paths:
  - "**/*.ps1"
  - "**/*.psm1"
  - "**/*.psd1"
---

# Windows PowerShell 5.1 Rules

**Applies when** the code targets Windows PowerShell 5.1 (`#Requires -Version 5.1`, manifest
`PowerShellVersion = '5.1'`, `CompatiblePSEditions = 'Desktop'`), **or** must run on both 5.1 and 7.
If the code targets PowerShell 7 only, ignore this file and follow `powershell-7-rules.md`.

All other rules files in this folder (`powershell-*-rules.md`) also apply.

## Declaring the Target

- Scripts: `#Requires -Version 5.1`
- Module manifest:

  ```powershell
  PowerShellVersion    = '5.1'
  CompatiblePSEditions = @('Desktop')            # or @('Desktop', 'Core') for dual-target
  RequiredModules      = @('PSFramework')
  ```

## Language Features That DO NOT Exist in 5.1

Never use these in 5.1-compatible code:

| PowerShell 7 feature | 5.1 alternative |
| --- | --- |
| Ternary `$a ? $b : $c` | `if ($a) { $b } else { $c }` |
| Null-coalescing `??` / `??=` | `if ($null -eq $x) { $x = $default }` |
| Null-conditional `?.` / `?[]` | Check `$null` explicitly first |
| Pipeline chain `&&` / `\|\|` | `if ($?) { ... }`, or `try` / `catch` |
| `ForEach-Object -Parallel` | `Start-ThreadJob` (install `Microsoft.PowerShell.ThreadJob`; the old `ThreadJob` name breaks on 7.6, so call `Start-ThreadJob` without a module qualifier), runspaces, or `PSFramework` runspace helpers. `Start-Job` starts a whole process per job, so use it only for isolation. |
| `clean {}` block | `try` / `finally` in `end {}` |
| `Join-Path` with more than 2 segments / `-AdditionalChildPath` | Nest `Join-Path` calls, or use `[IO.Path]::Combine()` |
| `Get-Error` | `$Error[0] \| Format-List * -Force` |
| `Test-Json` | Use `ConvertFrom-Json` inside `try` / `catch` |
| `ConvertFrom-Json -AsHashtable` / `-Depth` | Convert the `PSCustomObject` manually |
| `Invoke-RestMethod -SkipHttpErrorCheck` / `-StatusCodeVariable` / `-Authentication` | `try` / `catch` and read `$_.Exception.Response`, set auth headers manually |
| `$IsWindows`, `$IsLinux`, `$IsMacOS` | They don't exist. 5.1 is Windows-only. In dual-target code use `$onWindows = $PSVersionTable.PSVersion.Major -eq 5 -or $IsWindows`. |

Configure PSScriptAnalyzer to catch these:

```powershell
PSUseCompatibleSyntax = @{ Enable = $true; TargetVersions = @('5.1') }   # add '7.4' (or later) when dual-targeting
```

Also turn on `PSUseCompatibleCommands` and `PSUseCompatibleTypes` with a Windows PowerShell 5.1 target profile.

## File Encoding

- Save `.ps1`, `.psm1`, and `.psd1` files as **UTF-8 with BOM** if they contain any non-ASCII character. Without a BOM, 5.1 reads them as the system ANSI code page. PSScriptAnalyzer rule: `PSUseBOMForUnicodeEncodedFile`.
- Always pass `-Encoding` explicitly to `Out-File`, `Set-Content`, `Add-Content`, and `Export-Csv`. 5.1 defaults differ between cmdlets (UTF-16LE for `Out-File` and `>`, ANSI for `Set-Content`).
- In 5.1, `-Encoding UTF8` writes a BOM. If you need UTF-8 **without** a BOM, use `[IO.File]::WriteAllText($path, $text, [Text.UTF8Encoding]::new($false))`.
- Don't use the `>` / `>>` redirection operators to write files. Use `Out-File -Encoding ...`.

## Web & Networking

- Before making HTTPS calls, make sure TLS 1.2 is enabled:

  ```powershell
  [Net.ServicePointManager]::SecurityProtocol = [Net.ServicePointManager]::SecurityProtocol -bor [Net.SecurityProtocolType]::Tls12
  ```

  Do this once, in the module's `.psm1` or the script entry point, and log it with `Write-PSFMessage -Level Verbose`.
- Pass `-UseBasicParsing` to `Invoke-WebRequest`. This avoids the dependency on Internet Explorer's engine, which may not be available.
- `ConvertTo-Json` defaults to `-Depth 2` and silently truncates deeper objects. Always pass `-Depth` explicitly.

## Platform & Compatibility

- Use CIM cmdlets (`Get-CimInstance`, `Invoke-CimMethod`), not the WMI cmdlets (`Get-WmiObject` and the rest). WMI cmdlets don't exist in 7, so avoiding them keeps upgrades simple.
- Classes and enums defined in a module are only visible to consumers through `using module <ModuleName>`, not `Import-Module`. Expose functionality through functions, and treat classes as internal unless consumers use `using module`.
- PowerShell classes can't be reloaded in the same session. Restart the session (or run Pester in a new process) after you change a class.
- Use `.NET Framework 4.x` APIs only. Don't call .NET Core / .NET 5+ APIs (for example `[System.Text.Json]` and newer `String` overloads).
- Write `New-Object` or `[Type]::new()` consistently. `::new()` works in 5.1 and is preferred.
- `Group-Object` orders groups differently in 5.1 and 7. Sort explicitly (`Sort-Object -Property Name`) before relying on group order.
- .NET Framework's `[IO.Compression.ZipFile]::CreateFromDirectory()` writes entry names with `\` separators, which other platforms read as part of the file name. Use `Compress-Archive`, or build entries yourself with `/` separators.
- In 5.1, `HttpStatusCode` has no member for 429. Read status codes as integers (`[int]$_.Exception.Response.StatusCode`), never by name.
- In 5.1, the error response body can be read **once** from `$_.Exception.Response.GetResponseStream()`. Read it into a variable and dispose the reader in `finally`.
- In dual-target code, detect 7-only cmdlet parameters by **capability**, not version: `(Get-Command Invoke-WebRequest).Parameters.ContainsKey('SkipHttpErrorCheck')`. Detect once and cache it in `$script:` scope.

## Tooling on 5.1

- **Pester:** Windows ships with Pester 3.4, which is incompatible with these rules. Install Pester 6.2+ alongside it:

  ```powershell
  Install-Module -Name Pester -MinimumVersion 6.2.0 -Force -SkipPublisherCheck -Scope CurrentUser
  ```

  Then always `Import-Module Pester -MinimumVersion 6.2.0` in test runners.
- **PowerShellGet:** the built-in PowerShellGet 1.0.0.1 is outdated, and `Microsoft.PowerShell.PSResourceGet` isn't in-box on 5.1. Install PSResourceGet first, then use it. Bootstrap scripts that run on both editions detect it:

  ```powershell
  if (Get-Command -Name Install-PSResource -ErrorAction Ignore) {
      Install-PSResource -Name $moduleName -Scope CurrentUser -TrustRepository
  }
  else {
      Install-Module -Name $moduleName -Scope CurrentUser -Force -SkipPublisherCheck
  }
  ```

- **PSScriptAnalyzer, PSFramework, platyPS 0.14.x:** all support 5.1. Install them with `-Scope CurrentUser` unless you're told otherwise.
- Run the tests in `powershell.exe`, not just in `pwsh`, so they actually exercise 5.1 behavior.
