---
name: git-bash-editing
description: ALWAYS read before writing a script or a file body from Git Bash (never use a heredoc for file contents: use the Write and Edit tools). Rules for editing files from Git Bash on Windows without mangled backslashes, escapes that become real newlines, "unexpected EOF" rejections, CRLF patterns that match nothing, sqlcmd carriage returns, rewritten Windows paths, a working directory that resets, and junctions for scratch copies. Use when editing source files with sed/perl/python/heredocs, looping over sqlcmd output, or overwriting generated files that also hold hand edits.
---

# Editing files from Git Bash on Windows

## The rule

**Never carry a file body or a script in a Bash heredoc.** Use Bash to run things, not to carry contents.

| Need | Do this |
|---|---|
| Create or replace a file, or write a Python/JS script | **Write** tool (scripts to the scratchpad), then run `python C:/.../file.py` |
| Change one line or one short block | **Edit** tool, typing escapes exactly as they must appear in the file |
| The same replacement in many files | a script file made with Write: raw strings, `assert text.count(old) == 1`, read and write with `newline=''` |
| A short command with no `'`, `\`, backtick or `$` | Bash is fine |

A heredoc longer than about 5 lines, or holding any apostrophe, backslash, backtick or `$`, belongs in a file made with Write. After **every** scripted edit, `grep -n` or `sed -n` the changed line before building; a script that "ran" proves nothing.

## Shell rejections and rewrites

- **`unexpected EOF while looking for matching '`** rejects the whole command and nothing in it ran (check `ls -la` / `git status` before assuming a partial result). Cause: an apostrophe or quote inside a large heredoc, even with `<<'EOF'`. Fix: Write the file, then run it.
- **`rm -f $TEMP/*.py`** (a variable in a wildcard path) is refused and the whole command line, including the commands before it, does not run. Clean up in its own call with literal paths, or leave scratch files.
- **A Windows path passed to a Windows program is rewritten.** `sqlcmd -i /c/x/y.sql` fails with "Error occurred while opening ... C:" and the script silently did not run. Use the PowerShell tool with a backslash path, or `"$(cygpath -w /c/x/y.sql)"`. An argument starting with `/` is also turned into a path: set `MSYS_NO_PATHCONV=1`. After DDL, query the catalog to confirm it took effect.
- **Python run from Git Bash cannot see `/tmp`.** Put scratch files in the session scratchpad with a `C:/...` path.
- The **working directory resets** between Bash calls: use absolute paths or `cd` in the same command. Environment variables do not persist either: `export` them in the command that starts the server.

## Escapes: what reaches the file

- **Python/JS strings turn escapes into real characters.** `'\n'`, `'\r\n'`, `'\d'`, `'sql\functions'` (a form feed), `'8.0\bin'` (a backspace) written into a file come out as a real line break, a lone letter or a control character. Symptoms: C# `CS1010 Newline in constant`, `CS1009 Unrecognized escape sequence`, a regex `(\d{4})` arriving as `(d{4})`, a doc line printing `sqlunctions`. Python's `SyntaxWarning: invalid escape sequence` is the signal that the file content is now wrong.
  - Fixes in order: the Edit tool; a **raw** string `r'...'`; `chr(92)` for a backslash; in C# tests a verbatim string `@"...\d{4}..."`.
  - Through the Bash tool a doubled `\\n` in a Python heredoc arrives as `\n` and is then read as a newline. Another reason not to use heredocs.
  - `re.sub(pattern, "text with \n", s)` interprets backslashes in the replacement: pass a function (`lambda m: text`).
  - Never `sed 's/\\d/.../'` to fix a backslash: it matches every letter `d`.
  - After a scripted edit of a string literal, `grep -n` the line; a quote followed by a line break means repair it with Edit. For a corrupted path, `grep -c $'\x0c'` and `$'\x08'` find form feeds and backspaces.
  - A JS template literal turns `` \` `` into a backtick and `\\` into one backslash: write the content as it should appear, with no second layer.
- **`perl -e`, `perl -pi -e`, `sed s///` lose backslashes** (`'\\'` became `'\'`, `\.` lost its dot). In perl, write a single quote as `\x27` and a literal `$` as `\$`; better, avoid perl and use Edit. `sed -i` replacement text containing `|` or `&` needs escaping: beyond one literal word, use Edit.
- Verify regex edits by printing `repr()` of the text around a match before deciding the pattern is wrong.

## CRLF, BOM and line endings

- Most trees here are CRLF (`core.autocrlf=true`). A multi-line `perl -0pi` pattern with `\n` **matches nothing and reports no error**. Write `\r?\n`, emit `\r\n`, and check with `git diff`. The Edit tool also needs an exact match: on CRLF files edit single lines.
- In a Python script: `nl = '\r\n' if '\r\n' in s else '\n'`; read and write with `newline=''`; keep a UTF-8 BOM (`EF BB BF`, `utf-8-sig`) if the file had one; `assert s.count(old) == 1` before `.replace`. A file that mixes LF and CRLF: edit as bytes and match the line ending explicitly.
- Compare generated (LF) output with a CRLF original: `git diff -w --ignore-cr-at-eol --ignore-space-at-eol -U0 <files> | tr -d '\r' | grep '^[-+]'`. The "LF will be replaced by CRLF" warnings are harmless.
- The Edit and Write tools refuse a file not read in this conversation, or one changed since (by an IDE, a formatter, a sed pass): Read it again, then edit.

## sqlcmd and loops

- `sqlcmd` output ends each line with `\r`. `for t in $(sqlcmd ... -h -1 -W -Q ...)` then fails on every name but the last, and a `grep error` filter hides it. Pipe through `tr -d '\r'`, and when a loop produces fewer results than items, rerun one iteration unfiltered.

## Scratch copies and overwriting

- Copy a Node project without reinstalling: `tar --exclude=node_modules --exclude=.git --exclude=dist --exclude=.angular -cf - . | (cd DEST && tar xf -)`, then link `node_modules` with a junction made in **PowerShell**: `New-Item -ItemType Junction -Path DEST\node_modules -Target SRC\node_modules` (`cmd //c mklink /J` fails from Git Bash with "syntax incorrect").
- Before overwriting files that may hold hand edits, confirm `git status --short` is clean. `git checkout -- file` undoes a generate but also discards earlier hand edits in the same file: generate and hand-fix different files in different passes, or diff first.
- Program Files folders (such as the SQL Server Backup folder) may be unreadable and undeletable from the session: do not work around it with `xp_cmdshell` or bulk-delete procedures; tell the user the path.

## Process

- Read this skill **before** scripting edits; the rules were known and still skipped once.
- Do not `find /` for a file the user named: ask, or look only in folders the conversation mentioned.
- If a task is much bigger than its label ("small", "cheap"), say so and ask for the reduced or full version before starting.
- Real checks (an `npm install` and build per version, a database copy, a live API run) take minutes each: run them in one background job with a log file and a `DONE` marker, and read the log at the end.
- A fix found by a live run also needs a unit test, or the next edit loses it.
