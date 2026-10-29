---
name: powershell
description: Use when giving the user PowerShell commands to run themselves, or diagnosing a PowerShell or Windows terminal problem they report. Trigger words - line continuation, password prompt, working directory, multi-step instructions for Windows 11, Select-Object -First, quotes lost in a native command argument.
---

# PowerShell

A handful of gotchas from actually handing PowerShell commands to a Windows user mid-task, not a PowerShell tutorial.

## 1. Bash-style line continuation breaks

**Symptom:** a multi-line command using a trailing `\` fails with something like `Missing expression after unary operator '--'.`

**Cause:** PowerShell doesn't treat `\` as a line-continuation character.

**Fix:** use a trailing backtick (`` ` ``) for continuation, or just collapse the command to one line. Never hand a Windows/PowerShell user a bash-style `\`-continued multi-line command.

## 2. Interactive password prompts look hung

**Symptom:** a command that prompts `Enter password:` (e.g. `mysql -h host -u user -p`) appears to accept no input at all — no asterisks, no cursor feedback — from both Command Prompt and a PowerShell admin window. It isn't actually hung, but it reads that way and derails the task.

**Fix:** don't rely on the interactive prompt for anything scripted or handed to a user mid-task. Pass the credential through the environment for that command (`$env:MYSQL_PWD`, `$env:PGPASSWORD`, `$env:SQLCMDPASSWORD`), never in the argument list or a file (see the `mysql`, `postgres` and `mssql` skills). If the CLI tool isn't installed on the host, borrow one from a running container (`docker` skill, section 3) rather than asking the user to install it.

## 3. Working-directory assumptions silently break multi-step instructions

**Symptom:** a step assumes the user is still at the project root (e.g. `PS C:\project>`), but they're actually sitting inside a `mysql>` prompt or a different shell context left over from an earlier step, and the step fails in a confusing way.

**Fix:** when handing over a multi-step sequence, state the expected prompt/location at each step explicitly rather than assuming continuity from the previous one.

## 4. `| Select-Object -First N` stops the script upstream

**Symptom:** a script piped into `Select-Object -First 1` to keep the output short never finishes its work (a database copy was left half made: the backup was taken and the restore never ran).

**Cause:** `Select-Object -First N` ends the pipeline after N objects, which stops the commands that feed it.

**Fix:** keep the output short another way: redirect with `*> $null` (or `| Out-Null` only for commands that must run to the end), and check the result afterwards. Checked on 2026-10-29 in Windows PowerShell 5.1 and PowerShell 7: a marker written after the first output line was missing with `| Select-Object -First 1` and present with `*> $null`.

## 5. Windows PowerShell 5.1 drops double quotes inside native-command arguments

**Symptom:** `psql -c 'SELECT 1 AS "MixedCase"'` returns the column `mixedcase`, so a quoted mixed-case SQL identifier silently turns into a lower-case one.

**Cause:** Windows PowerShell 5.1 does not escape embedded double quotes when it builds the command line for an executable. PowerShell 7 does.

**Fix:** run the script in PowerShell 7 (`pwsh`), or send the text on stdin (`'SELECT 1 AS "MixedCase";' | psql ...`), which kept the quotes in both versions in the same test.

**See both problems in a shell:** run `C:\Github\tools\ps_quirks_check.ps1` with `powershell.exe -NoProfile -ExecutionPolicy Bypass -File ...` and with `pwsh -NoProfile -File ...` and compare (it needs a working `python` for the quoting part). Tested on 2026-10-29 in 5.1.26100 and 7.6.9.
