# skills

Personal [Claude Code](https://claude.com/claude-code) skills — concrete gotchas, conventions, and workflows
learned across real projects, not generic advice. Each skill is a `SKILL.md` (plus optional bundled
scripts/references/assets) that Claude Code loads automatically when its description matches the task at hand.

This repo is the version-controlled source of truth. Claude Code actually loads skills from
`C:\Users\ericw\.claude\skills\<skill-name>\` — a new or edited skill gets copied to both locations.

## Skills

### .NET / Desktop

| Skill | What it covers |
| --- | --- |
| [csharp](csharp) | Personal C# conventions — per-project `GlobalUsings.cs`, defaulting to modern C#/.NET syntax unless pinned to .NET Framework, single well-named boolean expressions over scattered conditionals |
| [winui3](winui3) | WinUI3/Windows App SDK gotchas — MVVM Toolkit partial-property build failures, TreeView binding limits, ContentDialog's single-open restriction, BitmapIcon vs ImageIcon, status icons via InfoBar, UI Automation-driven verification |
| [ui-conventions](ui-conventions) | Framework-agnostic desktop dialog/toolbar UX conventions — access keys, Escape/Enter, default-button focus, toolbar tooltips, colorblind-safe status icons |
| [msbuild](msbuild) | MSBuild/.csproj gotchas — XML-comment double-hyphen trap, culture-code false positives, RuntimeIdentifier singular-vs-plural, `.slnx` vs `.sln` |
| [self-contained-deployment](self-contained-deployment) | Copy-and-go C# desktop/CLI deployment philosophy — no installer, auto-seeded config, embedded assets |
| [codegen-t4-templates](codegen-t4-templates) | Writing/verifying T4 (Mono.TextTemplating) code-generator templates that reproduce hand-written files |
| [efcore-aspnet-review](efcore-aspnet-review) | Known defect patterns in EF Core entities and ASP.NET minimal-API CRUD endpoints |
| [bannerlord-decompile](bannerlord-decompile) | Decompiling a Bannerlord mod's DLLs with ILSpy and getting the result recompiling |

### Web

| Skill | What it covers |
| --- | --- |
| [angular-crud-gotchas](angular-crud-gotchas) | Angular (standalone components, Material, ng test/build) bugs in a CRUD screen talking to an ASP.NET API |

### Databases

| Skill | What it covers |
| --- | --- |
| [mssql](mssql) | Safety guardrails + gotchas for ad hoc T-SQL against SQL Server |
| [mysql](mysql) | Safety guardrails + gotchas for ad hoc SQL against MySQL |

### Deployment / DevOps

| Skill | What it covers |
| --- | --- |
| [docker](docker) | Docker/Docker Compose gotchas on a Windows host running Linux containers for a .NET/React app |
| [aws-lightsail-container-deploy](aws-lightsail-container-deploy) | Deploying multi-container apps to AWS Lightsail Container Service |

### Tooling / Workflow

| Skill | What it covers |
| --- | --- |
| [git-bash-editing](git-bash-editing) | Scripting file edits from Git Bash on Windows — heredoc/perl quoting, CRLF gotchas |
| [powershell](powershell) | Handing multi-step PowerShell instructions to a user on Windows |
| [difftastic](difftastic) | When and how to use `difft` instead of a plain line-based diff |
| [github-issue-creator](github-issue-creator) | Turning messy notes/error logs into a structured GitHub issue via `gh issue create` |
| [microsoft-docs](microsoft-docs) | Looking up current Microsoft Learn docs instead of relying on stale training data |
| [verify-in-running-app](verify-in-running-app) | Proving a change works by running the app against a disposable DB copy, not just passing tests |

### Meta

| Skill | What it covers |
| --- | --- |
| [skill-writer](skill-writer) | Creating and iterating on skills — writing guide, eval/benchmark loop, description optimization, packaging |

## Adding or updating a skill

1. Write/edit `<skill-name>/SKILL.md` here.
2. Copy the same skill folder to `C:\Users\ericw\.claude\skills\<skill-name>\` so Claude Code actually picks it up.
3. Update this README's table.
