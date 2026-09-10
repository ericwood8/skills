---
name: powershell
description: Windows/PowerShell terminal gotchas learned from handing multi-step CLI instructions to a user on a Windows 11 + PowerShell dev box — line continuation, interactive password prompts, and working-directory assumptions. Use when giving the user PowerShell commands to run themselves, or diagnosing a PowerShell/Windows-terminal execution problem they report.
---

# PowerShell

A handful of gotchas from actually handing PowerShell commands to a Windows user mid-task, not a PowerShell tutorial.

## 1. Bash-style line continuation breaks

**Symptom:** a multi-line command using a trailing `\` fails with something like `Missing expression after unary operator '--'.`

**Cause:** PowerShell doesn't treat `\` as a line-continuation character.

**Fix:** use a trailing backtick (`` ` ``) for continuation, or just collapse the command to one line. Never hand a Windows/PowerShell user a bash-style `\`-continued multi-line command.

## 2. Interactive password prompts look hung

**Symptom:** a command that prompts `Enter password:` (e.g. `mysql -h host -u user -p`) appears to accept no input at all — no asterisks, no cursor feedback — from both Command Prompt and a PowerShell admin window. It isn't actually hung, but it reads that way and derails the task.

**Fix:** don't rely on the interactive prompt for anything scripted or handed to a user mid-task — pass the credential inline instead, e.g. `mysql -h host -u user -p'password' ...` (no space after `-p`). If the CLI tool isn't installed on the Windows host at all, prefer borrowing it from an already-running container via `docker exec <container> <cli-command>` rather than asking the user to install a native Windows client.

## 3. Working-directory assumptions silently break multi-step instructions

**Symptom:** a step assumes the user is still at the project root (e.g. `PS C:\project>`), but they're actually sitting inside a `mysql>` prompt or a different shell context left over from an earlier step, and the step fails in a confusing way.

**Fix:** when handing over a multi-step sequence, state the expected prompt/location at each step explicitly rather than assuming continuity from the previous one.
