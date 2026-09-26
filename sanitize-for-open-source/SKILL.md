---
name: sanitize-for-open-source
description: Scrub employer/client/proprietary system names, personal Windows user-folder paths (C:\Users\<name>\..., C:\<PersonalWorkFolder>\...), real server/database names left in "verified live against..." testing narrative, and "ported from/adapted from/legacy of" lineage commentary out of a repo's README, docs/specs, code comments, and config before making it public on GitHub. Use when the user says a repo is going public, asks "any problems making X public", or asks to remove employer/company/personal references from code or documentation.
---

# Sanitizing a repo before it goes public

Personal preference behind this skill: **don't preserve the legacy of where an idea or piece of code
originally came from.** A doc comment or spec section that says "ported from `CompanyLib.Foo.Bar`" or
"adapted from the prior generator's `Whatever.cs`" is exactly the kind of thing to rewrite away — describe
what the code does and why, not its lineage or version history, unless that history carries real technical
information (a bug that was tried and rejected, an error number confirmed by testing, etc. — see the
distinction below).

## What to search for

Run a broad, case-insensitive search across the whole repo, excluding build output and `.git`:

```bash
grep -rniI "<employer-or-product-name>\|<personal-windows-username>" . \
  2>/dev/null | grep -v "/bin/" | grep -v "/obj/" | grep -v "/.git/"
```

Three independent things to look for in the same pass:

1. **Employer/client/product names** — anything naming a company, a proprietary product, or an internal
   system this repo's code was influenced by (`ported from CompanyLib.X`, `like ProductName's TableY`,
   `see the CompanyApp source`). These show up most often in:
   - Doc-comment headers on classes/methods that were modeled on something else
   - A "design lineage" / "prior art" / "reused code" section in a spec or design doc
   - Comments in test fixtures that name a real product's table/column as the inspiration for a sample
2. **Personal Windows folder paths** — `C:\Users\<name>\...`, or any absolute path revealing a personal or
   employer-internal folder layout (`C:\<PersonalWorkFolder>\...`). These show up in:
   - A spec's "source: `C:\...\`" line citing where code was reviewed from
   - Example CLI invocations that accidentally used a real local path instead of a placeholder
3. **Real server/database names in "verified live against..." testing narrative** — a status section, a
   changelog, or a README that documents real-world verification naturally wants to say *what* was tested
   against, and it's an easy habit to write the actual SQL Server instance name and a real client/company's
   database name straight into the sentence (`MYSERVER\ClientCompanyDb.dbo.SomeTable`). This is easy to
   reintroduce repeatedly rather than being a one-time leak: each new feature's own "generated live against
   the real database" writeup tends to be drafted fresh, copying the *pattern* of an earlier sentence
   without anyone re-checking whether the specific names in it are safe to publish. For a SQL Server-touching
   repo, grep for the `dbo.` schema prefix (a plain substring search — trying to also pattern-match the
   `SERVERNAME\` part reliably across different shells/tools is more fragile than it looks, backslash-
   escaping rules vary) and eyeball each hit for a real name in front of it:
   ```bash
   grep -rniI '\.dbo\.' . 2>/dev/null | grep -v "/bin/\|/obj/\|/.git/"
   ```
   Don't just delete the sentence — the fact that real-world verification happened is worth keeping; replace
   the specific server/database identity with a generic, consistent stand-in (see the rewrite example below)
   the same way a lineage comment gets reframed rather than deleted outright.

## What's fine to leave — don't over-scrub

- **The `LICENSE` copyright line** (`Copyright (c) <year> <github-username>`). This is the expected,
  required copyright holder attribution for an OSS license — removing it is a regression, not a fix. A
  GitHub username tied to the account publishing the repo is not a private-path leak.
- **The repo's own path used as a generic example**, e.g. `C:\Github\ThisRepo\Output` in a CLI usage
  example — that's just illustrating "wherever you cloned it," not leaking anything.
- **Genuinely generic placeholder paths** in usage examples (`C:\Work\MyApp\...`, `C:\MYSERVER`) — these are
  clearly stand-ins, not real personal paths.
- **A "second database"/"a real production database" style stand-in** already used consistently for a
  verification narrative (see the rewrite below) — that's the fix, not something left to finish.
- **Technical "why" history that isn't lineage**: "the first attempt did X, but SQL Server rejects Y (error
  264), confirmed by testing — fixed by Z" is valuable engineering history about *this* codebase's own
  design decisions, not a reference to an outside proprietary source. Keep that kind of comment; only cut
  the kind that names or points at somewhere else the code/idea came from.

## How to rewrite a lineage comment

Replace "where this came from" with "what this does." A three-part pattern that keeps the useful technical
content while dropping the attribution:

- Before: `/// <summary> Ported from CompanyLib.SqlServer.DataLayer.ColumnTools.IsAuditColumn. </summary>`
- After: `/// <summary> Name-pattern match for audit/tracking columns, used to exclude them from
  DisplayColumnSelector (Docs/specs.md section 6). </summary>`

For a whole "reused/ported code" section in a spec doc, don't just delete it — the *concepts* are often
still worth documenting (what behavior was deliberately reproduced, what was deliberately left out, and
why). Reframe it as a concept-level table ("Rich, cached table-model object" → what was re-derived here) 
instead of a source-file-by-source-file attribution table.

## How to rewrite a live-verification sentence naming a real server/database

Keep the technical claim (a real database, a real table shape, a real result); drop the specific identity.
Introduce one generic descriptor and reuse it for every sentence that named the same real server/database,
rather than inventing a different placeholder each time:

- Before: `` was generated live against `MYSERVER\ClientCompanyDb.dbo.Widgets` (correctly found ...) ``
- After: `` was generated live against a second database's `dbo.Widgets` table (correctly found ...) ``,
  with one earlier sentence establishing what "a second database" refers to ("a second, unrelated
  production database used for testing, on the same dev SQL Server").

The table/column names themselves (`Widgets`, `WidgetTypeID`) are usually fine to keep — they're a schema
*shape*, not an identity, unless a specific name is itself the giveaway (a column or table named after the
client's own product or internal team).

## Scope: working tree only, not git history

This skill only edits the current files (README, docs/specs, source comments, config) — it does **not**
touch git history. Old commits can still contain the pre-scrub text (or files later deleted, like an early
sample-output commit) even after the working tree is clean; anyone can `git show <old-commit>:<path>` to
recover them. That's a separate, genuinely destructive operation (`git filter-repo` + a force-push that
rewrites published commit hashes) — only do it if the user explicitly asks for it, and confirm first, since
it can't be cleanly undone and breaks anyone who already cloned/forked the repo. Don't bring it up as a
blocker unless the content in history is actually sensitive (credentials, real customer data) — a repo
owner may reasonably not care that old, already-superseded text is technically recoverable from history.

## Verify

After rewriting, re-run the search to confirm zero hits outside `.git`/`bin`/`obj`, then build and run the
test suite to confirm the edits (which are almost always comment/doc-only) didn't touch anything that
compiles or is asserted on.
