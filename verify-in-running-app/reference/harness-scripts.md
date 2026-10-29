# The harness is in the repository

Read this when: before you write a script for a scratch copy, UI Automation, paging or CRUD checks: the repository already has one.

`Docs/Verification` holds scripts for the above: `ScratchDatabase.ps1`, `UiAutomation.psm1` (`Start-App`, `Find-Name`, `Find-Like`, `Find-Type`, `Wait-Name`, `Invoke-El`, `Select-El`, `Set-Text`, `Assert-That`, `Stop-App`), `Test-AppDialogs.ps1`, `Test-ApiCrud.ps1`, `Test-WinUI3Paging.ps1`, `Test-AngularVersion.ps1` and a recipe per stack. Run them rather than rewriting the scaffolding. From Git Bash an argument starting with `/` is rewritten into a path: set `MSYS_NO_PATHCONV=1` when calling PowerShell. A scratch copy needs the right to create a database.
