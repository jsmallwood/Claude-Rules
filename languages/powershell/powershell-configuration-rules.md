---
paths:
  - "**/*.ps1"
  - "**/*.psm1"
  - "**/*.psd1"
---

# PowerShell Configuration Rules — PSFramework

Reference: [PSFramework — Configuration](https://psframework.org/docs/category/configuration)

Whenever a module or script needs configurable settings (endpoints, timeouts, paths, feature flags), use PSFramework
Configuration. Don't build your own JSON/XML/registry settings readers, and don't read settings from global variables.
Declare PSFramework in the module manifest's `RequiredModules`. Scripts declare it with `#Requires -Modules PSFramework`.

## Declaring settings (in the module)

Declare each setting once, at module import, in the module's `Configurations/*.ps1` files (see `powershell-modules-rules.md`):

```powershell
Set-PSFConfig -Module 'GitHubTools' -Name 'Api.BaseUri' -Value 'https://api.github.com' -Initialize -Validation 'string' -Description 'Base URI for the GitHub REST API.'
Set-PSFConfig -Module 'GitHubTools' -Name 'Api.PageSize' -Value 30 -Initialize -Validation 'integerpositive' -Description 'Number of items requested per page.'
Set-PSFConfig -Module 'GitHubTools' -Name 'Api.TimeoutSeconds' -Value 60 -Initialize -Validation 'integerpositive' -Description 'HTTP request timeout in seconds.'
```

- Always declare with `-Initialize`. That lets user-persisted values override the defaults.
- Always pass `-Validation` and `-Description`.
- Setting names are `<Module>.<Area>.<Setting>` with PascalCase segments (`GitHubTools.Api.BaseUri`).
- Use `-Handler { }` when changing a value needs to trigger something, such as clearing a cache.

## Reading settings

```powershell
$baseUri = Get-PSFConfigValue -FullName 'GitHubTools.Api.BaseUri'
$pageSize = Get-PSFConfigValue -FullName 'GitHubTools.Api.PageSize' -Fallback 30
```

- Read the value when you need it, inside the function, so runtime changes take effect. Don't store it in a module-level variable at import time.
- Use `-Fallback` only for settings that might not be declared.

## Persisting and importing settings

- Users persist settings with `Set-PSFConfig ... -PassThru | Register-PSFConfig` (per-user by default, `-Scope SystemDefault` for machine-wide).
- When settings ship as a file, load them with `Import-PSFConfig -Path <file.json>` and write them with `Export-PSFConfig`. Use the file format PSFramework produces. Don't invent your own schema.
- Never store secrets in PSFramework configuration. Use `Microsoft.PowerShell.SecretManagement`, and keep only the vault or secret *name* in configuration.
- Document every setting in the module's `about_<ModuleName>` help topic (see `powershell-documentation-rules.md`).

## Testing

- In Pester, set values with `Set-PSFConfig` in `BeforeAll`. Never edit registry or user config files from tests.
