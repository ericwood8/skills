---
name: difftastic
description: Use difftastic (the `difft` CLI) whenever comparing two files, two directories, or two versions of code/config/SQL and a plain line-based diff (git diff, fc, Compare-Object) would be noisy or misleading — especially when a large block was rewritten and a normal diff shreds it into dozens of scattered +/- lines instead of showing it as one coherent change. Trigger this on requests like "diff these two files", "compare this SQL script against the restored version", "show me what changed in this class", "write the differences to a text file", or any time you're about to reach for `git diff` on a real source/code/SQL file and want block-aware output instead of line noise. Also read this before invoking `difft` in a brand new session, since a just-installed or PATH-updated `difft` frequently fails to resolve in a fresh shell for reasons this skill explains and works around.
---

# difftastic (`difft`)

## What it is and why it's worth reaching for

difftastic is a **structural, syntax-aware** diff tool (built on tree-sitter grammars)
rather than a pure line-matcher. Classic diff algorithms (Myers, the default in `git
diff`, `fc`, `Compare-Object`, etc.) find the *minimal edit script* between two files
treated as flat lists of lines. That's great when changes are small and localized, but
when a large block of code is genuinely rewritten, the minimal-edit-script algorithm
often interleaves spurious matches across the block, producing a wall of scattered
single-line `+`/`-` pairs that obscures the one real change: "this whole function/query
was replaced."

difftastic instead parses each file into its syntax tree (when it recognizes the
language) and diffs at that structural level, so a rewritten block reads as one
coherent change. For files/languages it doesn't have a grammar for, it falls back to a
line-based diff — it never fails outright on an unrecognized file type.

It ships as a **single self-contained binary** (`difft` / `difft.exe`). There is no
separate per-language package to install — every bundled grammar (currently ~40+
languages including C#, SQL, Python, JS/TS, Go, Rust, and more) is compiled into that
one binary. Don't hardcode the language list in your own reasoning or in anything you
tell the user — it can grow between difftastic releases. Check what's actually bundled
in the installed binary at runtime:

```
difft --list-languages
```

## Step 1: Confirm `difft` actually resolves before using it

Do this at the start of any session where you're about to use difftastic, especially
if it was recently installed or updated — don't assume a prior session's success means
it still resolves now.

**Bash:**
```bash
which difft && difft --version
```

**PowerShell:**
```powershell
Get-Command difft -ErrorAction SilentlyContinue
difft --version
```

If either resolves cleanly, skip straight to Step 3 (Usage).

## Step 2: If `difft` is "not found" — diagnose before giving up

A "command not found" / "not recognized" result right after an install (via winget,
scoop, choco, or a manual PATH edit) almost always means the *installer's PATH change
already landed in the registry/user environment*, but **this process's shell inherited
its environment from its parent at spawn time** and hasn't picked up the change yet.
Restarting just a chat session/tab is usually not enough — the underlying host
application process itself needs to fully exit and relaunch (or the user needs to log
off/on) before a new child shell inherits the updated `PATH`. Diagnose rather than
assume:

**1. Check whether the install actually registered `PATH` correctly** (this reads the
persisted registry value, not this process's stale copy — so it tells you whether the
problem is "not installed" vs. "installed but not yet inherited"):

```powershell
[System.Environment]::GetEnvironmentVariable("PATH","User")   -split ';' | Select-String -Pattern 'difftastic'
[System.Environment]::GetEnvironmentVariable("PATH","Machine") -split ';' | Select-String -Pattern 'difftastic'
```

- **Found in one of these** → it *is* installed and `PATH` *is* correct; this
  process's shell is just stale. Tell the user plainly that a full restart of the
  host application (not just the current session/tab) — or a log off/on — is needed
  for new shells to see it. In the meantime, use the direct-invocation workaround
  below so you can keep working right now.
- **Not found anywhere** → it may genuinely not be installed, or was installed to a
  location that never touched `PATH` (e.g. a portable zip extract). Locate the binary
  directly (next) before concluding it's missing.

**2. Locate the binary directly, generically** (never hardcode a specific user's home
directory — always resolve through the environment variable so this works on any
machine):

```powershell
Get-ChildItem -Path "$env:LOCALAPPDATA\Microsoft\WinGet\Packages" -Directory -ErrorAction SilentlyContinue |
    Where-Object { $_.Name -like '*difftastic*' -or $_.Name -like '*Wilfred*' }
```

If that turns up a package directory, look inside it for `difft.exe`:

```powershell
Get-ChildItem -Path "$env:LOCALAPPDATA\Microsoft\WinGet\Packages" -Recurse -Filter "difft.exe" -ErrorAction SilentlyContinue
```

Also worth checking if winget doesn't turn up anything: a scoop install lives under
`$env:USERPROFILE\scoop\apps\difftastic\`, and a choco install typically ends up
already on `PATH` via a shim in `$env:ChocolateyInstall\bin`.

**3. Invoke it directly by the discovered path** as a workaround until the shell
environment refreshes — don't wait on a restart if the user wants to keep working now:

```powershell
$difft = (Get-ChildItem -Path "$env:LOCALAPPDATA\Microsoft\WinGet\Packages" -Recurse -Filter "difft.exe" -ErrorAction SilentlyContinue | Select-Object -First 1).FullName
& $difft --version
```

In Bash, the equivalent is just calling the full Windows path you found (Git Bash
accepts either `C:\...` or `/c/...` style paths):

```bash
"/c/Users/$USER/AppData/Local/Microsoft/WinGet/Packages/"*difftastic*/difft.exe --version
```

(`$USER`/`$USERNAME` here is the shell's own env var for whoever is running it — still
not a hardcoded name.)

If you search and still find nothing anywhere, say so plainly rather than guessing —
report that `difft` doesn't appear to be installed and ask whether to install it
(`winget install Wilfred-Hughes.difftastic`) rather than silently falling back to a
different diff tool without saying why.

## Step 3: Usage

Once `difft` resolves (directly or via `PATH`):

```bash
# Two files, printed to the terminal (default: side-by-side display)
difft old.sql new.sql

# Write a clean diff to a text file — color codes must be disabled, or the
# file fills with ANSI escape sequences instead of readable text
difft --color=never old.cs new.cs > diff.txt

# Two directories — diffs matching filenames pairwise
difft dir_old dir_new

# Narrower/inline display instead of side-by-side — useful when the output
# needs to stay readable in a narrow terminal or an agent transcript
difft --display=inline old.cs new.cs

# One-off use as git's diff driver, without changing global git config
git -c diff.external=difft diff
GIT_EXTERNAL_DIFF=difft git diff   # equivalent, POSIX shells
```

## When to reach for difftastic vs. a plain line diff

- **Prefer difftastic** for actual source/code/config/SQL files, especially when you
  expect (or the user reports) that a large chunk was rewritten — that's exactly the
  case its structural diffing handles better than line-based tools.
- **Prefer `git diff --histogram` (or `--patience`)** as the zero-install fallback for
  plain text, or when difftastic genuinely isn't available and installing it isn't an
  option in the moment. These algorithms anchor on unique matching lines first, which
  also reduces (though doesn't eliminate, the way structural diffing does) the
  "shredded diff" problem versus the default Myers algorithm:
  ```bash
  git diff --no-index --histogram --color=never old.txt new.txt > diff.txt
  ```
- difftastic falls back to line-based diffing itself for any file type it doesn't have
  a grammar for, so it's safe to reach for by default on any two-file comparison — it
  won't error out on an unrecognized extension, it just won't get the structural
  benefit.
