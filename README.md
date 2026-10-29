# skills

Personal [Claude Code](https://claude.com/claude-code) skills — concrete gotchas, conventions, and workflows
learned across real projects, not generic advice. Each skill is a `SKILL.md` (plus optional bundled
scripts/references/assets) that Claude Code loads automatically when its description matches the task at hand.

This repo is the only copy. Claude Code loads skills from `C:\Users\<UserName>\.claude\skills`, which is a
directory junction to this folder (`mklink /J C:\Users\<UserName>\.claude\skills <path to this repo>`), so an
edit here is live at once and nothing is copied.

## Skills

### .NET / Desktop

| Skill | What it covers |
| --- | --- |
| [csharp](csharp) | Personal C# conventions — per-project `GlobalUsings.cs`, defaulting to modern C#/.NET syntax unless pinned to .NET Framework, single well-named boolean expressions, `required`/`init` over mutable setters, records for DTOs, propagating `CancellationToken` through async chains |
| [winui3](winui3) | WinUI3/Windows App SDK gotchas — MVVM Toolkit partial-property build failures, TreeView binding limits, ContentDialog's single-open restriction, BitmapIcon vs ImageIcon, status icons via InfoBar, UI Automation-driven verification |
| [ui-conventions](ui-conventions) | Framework-agnostic desktop dialog/toolbar UX conventions — access keys, Escape/Enter, default-button focus, toolbar tooltips, colorblind-safe status icons |
| [msbuild](msbuild) | MSBuild/.csproj gotchas — XML-comment double-hyphen trap, culture-code false positives, RuntimeIdentifier singular-vs-plural, `.slnx` vs `.sln` |
| [self-contained-deployment](self-contained-deployment) | Copy-and-go C# desktop/CLI deployment philosophy — no installer, auto-seeded config, embedded assets |
| [codegen-t4-templates](codegen-t4-templates) | Writing/verifying T4 (Mono.TextTemplating) code-generator templates that reproduce hand-written files |
| [odata-generated-grids](odata-generated-grids) | CodeGenNew OData/REST grids in React, Angular and Blazor: `$select` from chosen columns, dropping sorts and filters with a hidden column, owner scope, and how to prove a grid change live |
| [efcore-aspnet-review](efcore-aspnet-review) | Known defect patterns in EF Core entities and ASP.NET minimal-API CRUD endpoints |
| [bannerlord-decompile](bannerlord-decompile) | Decompiling DLLs with ILSpy and getting the result recompiling |
| [mstest-assertions](mstest-assertions) | Fixing MSTEST0037 by using the specific MSTest assertion (`Assert.Contains`, `HasCount`, `IsLessThan`), expected value first |

### Delphi

| Skill | What it covers |
| --- | --- |
| [delphi](delphi) | Delphi / Object Pascal language and RTL guide: version gating, memory and lifetime, strings, exceptions, generics and RTTI, concurrency, style |
| [delphi-code-smells](delphi-code-smells) | Reviewing Delphi code: compiler warnings that mean a bug, leak detection, static analysis |
| [delphi-types-and-units](delphi-types-and-units) | Class anatomy, virtual/override/abstract, records, helpers, operators, interfaces, enums and sets, pointers, units, .dpr and .dproj |
| [delphi-firedac](delphi-firedac) | FireDAC: connection, queries, parameters, transactions, threads, for SQLite, SQL Server, MySQL and PostgreSQL |

`delphi` and `delphi-code-smells` come from [danieleteti/delphi-ai-skills](https://github.com/danieleteti/delphi-ai-skills) (Apache License 2.0; the license text is in [THIRD_PARTY_LICENSES](THIRD_PARTY_LICENSES)). They are the upstream files except that every DMVCFramework reference was removed (the framework is not used here). `delphi-types-and-units` and `delphi-firedac` are written here and were not compiled (the installed Delphi is the Community edition, which cannot compile from the command line).

### Web

| Skill | What it covers |
| --- | --- |
| [angular-crud-gotchas](angular-crud-gotchas) | Angular (standalone components, Material, ng test/build) bugs in a CRUD screen talking to an ASP.NET API |
| [react-crud-gotchas](react-crud-gotchas) | React (Vite, react-router) bugs in a generated CRUD screen app: form under a long grid, hidden-tab validation, clearing a date, child-row navigation |

### Databases

| Skill | What it covers |
| --- | --- |
| [mssql](mssql) | Safety guardrails + gotchas for ad hoc T-SQL against SQL Server, exporting schema and data with bcp |
| [mysql](mysql) | Safety guardrails + gotchas for ad hoc SQL against MySQL |
| [postgres](postgres) | Safety guardrails + gotchas for ad hoc SQL against PostgreSQL, reading its catalog, functions vs procedures, Npgsql |

### Deployment / DevOps

| Skill | What it covers |
| --- | --- |
| [docker](docker) | Docker/Docker Compose gotchas on a Windows host running Linux containers for a .NET/React app |
| [aws-lightsail-container-deploy](aws-lightsail-container-deploy) | Deploying multi-container apps to AWS Lightsail Container Service |
| [nginx-config-testing](nginx-config-testing) | nginx config for a built Angular/React front end (gzip, hashed-file caching, keep-alive to the API), proved with the real nginx image |

### Tooling / Workflow

| Skill | What it covers |
| --- | --- |
| [git-bash-editing](git-bash-editing) | Scripting file edits from Git Bash on Windows — heredoc/perl quoting, CRLF gotchas |
| [powershell](powershell) | Handing multi-step PowerShell instructions to a user on Windows |
| [difftastic](difftastic) | When and how to use `difft` instead of a plain line-based diff |
| [github-issue-creator](github-issue-creator) | Turning messy notes/error logs into a structured GitHub issue via `gh issue create` |
| [microsoft-docs](microsoft-docs) | Looking up current Microsoft Learn docs instead of relying on stale training data |
| [verify-in-running-app](verify-in-running-app) | Proving a change works by running the app against a disposable DB copy, not just passing tests |
| [minimal-comments-and-docs](minimal-comments-and-docs) | Writing less: no section or page numbers in comments, no restating the code, no double hyphens or em dashes, named constants for error numbers, no counts in docs |
| [sanitize-for-open-source](sanitize-for-open-source) | Scrubbing employer names, personal paths and real server or database names from a repo before it goes public |

### Meta

| Skill | What it covers |
| --- | --- |
| [skill-writer](skill-writer) | Creating and iterating on skills — writing guide, eval/benchmark loop, description optimization, packaging |

## Adding or updating a skill

1. Write/edit `<skill-name>/SKILL.md` here.
2. Update this README's table.
