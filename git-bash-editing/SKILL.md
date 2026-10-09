---
name: git-bash-editing
description: Gotchas learned scripting file edits from Git Bash on Windows — backslashes and quotes mangled when code is passed through heredocs or perl one-liners, perl multi-line replacements silently matching nothing on CRLF files, sqlcmd output carrying stray carriage returns into shell loops, escapes written through Python strings becoming real newlines, "unexpected EOF while looking for matching quote" rejecting a command with a large heredoc, Windows paths rewritten for sqlcmd -i (use PowerShell or cygpath -w), a working directory that resets, and node_modules junctions for scratch copies. Use when editing source files with sed/perl/heredocs/node scripts, looping over sqlcmd output, or overwriting generated files that also hold hand edits.
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

## Four failures that kept recurring in one long session (and the rule for each)

1. **"The backslash escaping slipped again" — an escape written through Python became a real character.** A Python script (run from a Bash heredoc or
   `python - <<'EOF'`) that writes C# or JavaScript containing `\n`, `\r\n`, `\d`, `\\` inside an ordinary `'...'` or `'''...'''` string writes a **real
   line break / lone backslash** into the file, not the two characters the target language needs. Symptoms: C# `CS1010 Newline in constant`, `CS1009 Unrecognized
   escape sequence`, a regex such as `(\d{4})` arriving as `(d{4})`. Python itself warns first (`SyntaxWarning: "\d" is an invalid escape sequence`) — treat that
   warning as the signal that the file content is now wrong. Fixes, in order of preference:
   - **Edit** the one line with the Edit tool (type the escapes as they must appear in the file).
   - Put the code in a Python **raw** string (`r'''...'''`) so backslashes stay as written; or build the backslash with `B = chr(92)`.
   - In C# tests that check generated regex text use a verbatim string (`@"...\d{4}..."`) so the test file needs no doubled backslashes.
   - **`re.sub(pattern, "text with \n", s)` also interprets backslashes in the replacement**; pass a function (`lambda m: text`) instead.
   - **Never `sed 's/\\d/.../'` to fix a backslash**: it matched every letter `d` (`</td>` became `<t\d>`, `child.Var` became `chil\d.Var`). Do the replacement in Python with an
     exact string, or use Edit.
2. **"The newline slip in that Parse call" — the same cause in a test.** A line such as `ProjectSettings.Parse("ProjectName=Acme\nEnumTables=none")` written through
   a Python string came out as a string literal split across two lines. After any scripted edit of a C# string literal, `grep -n` the line and look; if a quote is
   followed by a line break, repair that line with Edit. Prefer test helpers that avoid newlines in literals (`FromValues(...)` with key/value pairs).
3. **`/usr/bin/bash: -c: line N: unexpected EOF while looking for matching '` rejects the whole command — nothing in it ran.** Seen twice with a command that held
   a large heredoc (a SQL file or a Python script) followed by more commands. Check what changed (`ls -la`, `git status`) before assuming a partial result:
   it was zero. The reliable fix is to **stop putting file bodies in heredocs**: create the file with the **Write** tool, then run it (`python file.py`,
   `sqlcmd -i file.sql`). Heredocs are fine only for short bodies with no apostrophes, backslashes or `$`.
4. **A Windows path passed to a Windows program from Git Bash is rewritten.** `sqlcmd -i /c/InvoiceSystem/sql/x.sql` (and `-i "C:/..."`) fails with
   `Sqlcmd: Error: Error occurred while opening or operating on file C: (Reason: Access is denied)` — and the script silently did not run (the next query shows the
   unchanged database). Run the program with the **PowerShell tool** and a backslash path, or pass `"$(cygpath -w /c/path/x.sql)"` from Bash. After running any DDL, query the
   catalog to confirm it took effect (column counts, `sys.foreign_keys`, `COL_LENGTH`) instead of trusting a silent exit.


## More traps seen while patching templates and docs with scripts

- **A heredoc that contains an apostrophe can end the command early** ("unexpected EOF while looking for matching `'`") even with a quoted `<<'EOF'` when the Bash tool wraps the command; a long script (or a multi-file `cat > ... <<EOF` batch) with quotes in comments or strings is safer written with the **Write tool** (to the scratchpad) and then run with `python file.py` / copied into place.
- **Escape sequences in a Python (or JS) string quietly become control characters**: `'sql\functions'` contains a **form feed** (`\f`), `'8.0\bin'` a **backspace** (`\b`), `'\t'`, `'\n'` real tabs/newlines. A path like `C:\InvoiceSystem\bin` in a non-raw string written into a doc or spec is corrupted without any error (a doc line printed `sqlunctions`). Use raw strings (`r'...'`), `chr(92)` for a backslash when building a pattern, or the Edit tool; afterwards `grep -c $'\x0c'` / `$'\x08'` the file.
- A JS template literal turns `\`` into a backtick and `\\` into one backslash: when writing markdown with backticks and Windows paths, write the content in the template literal exactly as it should appear and do not add a second layer of escaping.
- When a Python `.replace(old, new)` must match exactly once, `assert u.count(old) == 1` first and normalise `\r\n` to `\n` before and back after (`nl = '\r\n' if '\r\n' in s else '\n'`); read and write with `newline=''` so the file's own line endings survive, and keep a UTF-8 BOM if the file had one (`utf-8-sig`).

## Large heredocs and scripts: more traps (2026-10)

- A big `python - <<'EOF'` or `cat > file <<'EOF'` that holds backslashes, backticks and quotes together was twice **mangled or rejected** by the shell tool (`unexpected EOF`, a backslash eaten: `'\Search'` lost its backslash, a C# `"\"` became `""`). For any file or script with such text, use the **Write** tool (to the scratchpad for a script) and run it, or the Edit tool.
- Regex replacements over files that contain Windows paths or `\` need `chr(92)` or a raw-string check; verify by printing `repr()` of the text around a match before concluding the pattern is wrong.
- `rm -f $TEMP/*.py` is refused (a variable in a wildcard path) and **the whole command is not run**, including the commands before it in the same line. Put clean-up in its own call, use literal paths, or leave scratch files.
- Git Bash turns an argument starting with `/` into a Windows path when it calls a native program (`MSYS_NO_PATHCONV=1` stops it).
- `sed -i` on a file whose line contains `|` or `&` inside the replacement needs escaping; for anything beyond one literal word use the Edit tool.
- Preserve BOM and line endings when a script rewrites a file: read bytes, remember `startswith(b'\xef\xbb\xbf')` and whether `\r\n` was present, write them back.
- Python run from Git Bash cannot see `/tmp`; write scratch files to the session scratchpad with a `C:/...` path. When a file mixes LF and CRLF lines, edit it as bytes and match `` explicitly instead of assuming one ending.

## Backslashes lost in sed and perl replacements; the Edit tool wants a fresh Read

- A replacement that must contain `\.` (an nginx or other regex) lost its backslash through `perl -e` and `sed s///` run from Git Bash, twice in a row, and the file looked fine until it was read. For any text with backslashes use the Edit tool (or Write), then grep the line to confirm.
- The Edit and Write tools refuse a file that was not read in this conversation, and also one that another process (an IDE, a formatter, a linter) changed since the last read: read the lines again, then edit. Expect this after a sed/perl pass on the same file.
- A multi-line `perl -0pi -e` with a long pattern silently matched nothing; check with `git diff` or `sed -n` after every such edit, and fall back to the Edit tool.
- A very large heredoc to create a file can fail with "unexpected EOF while looking for matching quote" and write nothing; use the Write tool for file contents and keep Bash for running things.
