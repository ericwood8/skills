---
name: git-bash-editing
description: Gotchas learned scripting file edits from Git Bash on Windows — backslashes and quotes mangled when code is passed through heredocs or perl one-liners, perl multi-line replacements silently matching nothing on CRLF files, sqlcmd output carrying stray carriage returns into shell loops, a working directory that resets, and node_modules junctions for scratch copies. Use when editing source files with sed/perl/heredocs/node scripts, looping over sqlcmd output, or overwriting generated files that also hold hand edits.
---

# Editing files from Git Bash on Windows

## Backslashes and quotes: use the Edit/Write tools for code that contains them

- Code passed through `cat <<'EOF'`, `perl -e`, or `perl -pi -e` **loses or mangles backslashes**. Seen twice in one session: a C# line `path.Replace('\\', '/')` was written to disk as `path.Replace('\', '/')` (compile error: "Too many characters in character literal"), and a JavaScript line testing `c === '\\'` became `'\'`. A second attempt with a different escape style failed the same way.
- **Rule:** any file whose content has `\\`, `\n` inside string literals, regex escapes, or Windows paths → create it with the **Write** tool and change lines with the **Edit** tool, not a heredoc or a perl substitution. Use `String.fromCharCode(92)` / `Path.DirectorySeparatorChar` only as a last resort.
- In a perl substitution inside single quotes, write a single quote as `\x27` and a literal `$` as `\$`; better still, avoid perl for that edit.
- After **every** scripted edit, look at the result (`sed -n 'N,Mp'`, `grep -n`, or `git diff`) before building. A script that "ran" proves nothing.

## perl -0pi on CRLF files fails silently

- Most working trees here are CRLF (`core.autocrlf=true`). A multi-line pattern written with `\n` **matches nothing and reports no error**, so the file is left unchanged and you go on believing it changed. Write line breaks as `\r?\n` and emit replacements with `\r\n` (or rely on git normalizing).
- The Edit tool needs an exact match too; on CRLF files prefer perl with `\r?\n`, or edit single lines.
- Preserve a UTF-8 BOM if the file has one (the first bytes `EF BB BF`); tools that rewrite the file may drop it. Generated files usually lose it — mention it, it shows as a line-1 diff.
- `git diff -w --ignore-cr-at-eol --ignore-space-at-eol -U0 <files> | tr -d '\r' | grep '^[-+]'` shows only the real changes when comparing generated (LF) output to a CRLF original; the "LF will be replaced by CRLF" warnings are harmless.

## sqlcmd output has carriage returns

- `for t in $(sqlcmd ... -h -1 -W -Q "SELECT name ...")` yields names ending in `\r`. Every command using them then fails ("table not found") — and if you filter the loop's output with `grep` for "error", the failures are **invisible**: only the last item (no trailing `\r`) worked. Pipe through `tr -d '\r'`, and when a loop "succeeds" but produces fewer results than items, rerun one iteration unfiltered.

## Shell state

- The working directory **resets between Bash calls**; use absolute paths or `cd` in the same command. Environment variables also do not persist — `export` them in the same command that starts a background server.
- Program Files (for example the SQL Server Backup folder) may be unreadable and undeletable from the assistant's session: `ls`, `rm` and PowerShell `Remove-Item` all fail with "denied". Don't work around it with `xp_cmdshell` or bulk-delete procedures; tell the user the path to delete.
- A scratch copy of a Node project without re-installing: `tar --exclude=node_modules --exclude=.git --exclude=dist --exclude=.angular -cf - . | (cd DEST && tar xf -)`, then a junction: `cmd //c "mklink /J node_modules C:\path\to\node_modules"`.

## Before overwriting files that may hold hand edits

- Confirm the repo is clean (`git status --short`) or that the user pushed a backup, then overwrite. `git checkout -- file` is the undo, but it also discards **any earlier hand edit in the same file** — so generate and hand-fix different files in different passes, or diff first.
