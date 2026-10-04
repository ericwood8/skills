---
name: minimal-comments-and-docs
description: House rules for code comments, file headers and documentation in any repository. Use whenever writing or editing comments, XML doc comments, template headers, README or design docs, or when asked to document code. Covers no section or page numbers, no comments that restate code, no double hyphens or em dashes outside real SQL, comments that state purpose, named constants for error numbers, and keeping documentation minimal.
---

# Comments and documentation: say less, say why

The process is agile: working code and tests are the documentation, and written text is kept small because every extra line is a line that goes stale. Apply these rules to every comment, header, doc comment and document you write or touch.

## 1. No section, page, item or line numbers

Never cite "section 5.3", "item 61", "page 4", "round 2 Q1" or a line number in a comment or doc. Documents get renumbered, rewritten and archived, and the citation then points at nothing (a past cleanup had to retarget about 115 such comments). Name the thing instead: the type, the file, the setting, or the rule in words ("see ARCHITECTURE.md", "the `.tt.config` keys in Reference.md"). If the comment already states the rule, drop the citation entirely.

Also leave out where an idea came from (who asked, which round, which earlier tool). Describe current behaviour and why.

## 2. Do not repeat what the code says

A comment earns its place only by telling what a reader **cannot work out by reading the code**: the reason, the constraint, the trap, the unit, the consequence of getting it wrong.

- Delete comments that restate the next line, the method name, or the type name (`// loop over the rows`, `/// <summary> Gets the name. </summary>` on `Name`).
- **File headers especially:** no banner with the file name, "Generates: ...", author, date or a list of what is in the file. The file name and the code already say it. Keep a header only for something non-obvious that applies to the whole file (an assumption another file must honour, a safety rule), in one or two sentences.
- No change logs in comments; git has the history.
- Do not add XML doc comments to every member. Add them where the contract is not obvious from the signature (units, null meaning, thrown errors, side effects).

## 3. Purpose and intent, especially for unreadable code

Say **why**, not what. This matters most where the code cannot explain itself:

- **Regular expressions:** one line saying what they must match and what they must not (`// a version like 1.2 or 1.2.3, not a date`). Name the pattern (`VersionPattern`) rather than inlining it.
- Bit masks, magic arithmetic, SQL tricks, unusual ordering, a workaround for a library or database quirk (name the quirk and the symptom).
- A deliberate omission ("no retry here: the caller owns it").

If you cannot state the purpose in a sentence, the code probably needs renaming or splitting; fix that first and the comment often disappears.

## 4. No double hyphens or em dashes outside real SQL

Do not write `--`, `---` or the em dash (—) or en dash (–) as punctuation in comments, docs, commit text or messages. Rephrase with a comma, colon, parentheses or two sentences. Use a plain hyphen only inside a compound word.

Exceptions: `--` that is genuinely SQL comment syntax or a command-line flag (`--dry-run`) inside code or a code block. In generated or templated text that becomes XML, XAML or HTML comments, a `--` is illegal and breaks the compiler, so a template must build any needed run of hyphens from characters instead of writing them literally.

## 5. Named constants for error numbers and magic values

Do not leave `547`, `-1`, `1451`, `0x80070005` or `"23503"` bare. Give each a constant whose name says the meaning, and use the constant. The number then needs no comment.

```csharp
private const int SqlServerForeignKeyViolation = 547;
private const int DeleteBlockedByDependency = -1;
```

In SQL and generated text where a constant is not possible, put the meaning once beside the value, in the same file. Same rule for timeouts, sizes, limits and status codes: a named constant with its unit in the name (`RetryDelayMilliseconds`).

## 6. Minimal documentation overall

- Prefer making the code, names and tests clear over writing about them.
- One short README (what it is, how to run and test, where to look next) and, if the system is large, one short architecture guide. Do not duplicate the same facts in several documents; link instead.
- Write a design document only when a decision needs a record, keep it short, and state the decision and the reason, not the history.
- Do not write a document, summary file or notes file unless asked. Tests that show behaviour beat prose that describes it.
- When code changes, fix or delete the comment beside it in the same edit. A wrong comment is worse than none.

## Quick check before finishing an edit

1. Any digit after the word section, item, page, step or line? Remove it.
2. Does each comment say something the code does not? If not, delete it.
3. Any `--` or em dash in prose? Rephrase.
4. Any regex or odd expression without a purpose line? Add one.
5. Any bare error number? Name it.

## Scope and exceptions (decided by the owner)

- **New files get the rules from the start. Existing files are not edited just to fix comments.** If you are changing a file for another reason, apply the rules to the lines you touch and the comments beside them, but do not sweep the rest of the file, and do not open a file only to reword a comment. Never make a repository-wide comment cleanup unless asked.
- **The one standing sweep is bare error numbers** (rule 5): find them across the code and name them, since that is a code-quality fix and not a comment edit.
- **A spaced hyphen is fine** (` - ` between words). Rule 4 bans double hyphens and em or en dashes, not a single hyphen.
- **Text that a generator writes into the user's own files is an exception to rule 2.** Generated code is a skeleton the user will extend, and they did not write it, so explanatory comments and headers in generated output are welcome (what the file is for, what is safe to edit, what a rule does and why). Keep them useful and short, but do not strip them for being "obvious to the author". Rules 1, 3 (no numbered citations), 4 (no double hyphens or em dashes in prose, except genuine SQL comments) and 5 still apply to generated text, and changing generated text changes the output of every regenerated project, so treat it as an output change (dry-run and review the diff).
