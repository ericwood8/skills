---
name: github-issue-creator
description: Turn messy input (error logs, voice-dictated bug reports, screenshots, casual descriptions) into a clean, structured GitHub issue and open it with `gh issue create`. Use when the user pastes an error, describes a bug informally, or asks to file/open a GitHub issue from notes.
---

# GitHub Issue Creator

Turn messy input into a crisp GitHub issue, then open it with `gh` — don't just draft a file nobody reads.

## Process

1. **Extract structure from chaos.** Pull reproduction steps, error text, and expected-vs-actual behavior out of casual language, even from a stream-of-consciousness voice note or a raw stack trace paste.
2. **Infer missing context** from the conversation or repo (project name, affected file, recent commit, `git remote`) instead of asking the user to restate what's already known — but don't invent details nothing implies.
3. **Draft the issue** with the template below and show it to the user before creating anything.
4. **Create it** with `gh issue create --repo <owner/repo> --title "..." --body "$(cat <<'EOF' ... EOF)"` only after the user confirms the draft. Opening an issue is visible to everyone watching the repo, so don't skip the confirmation step even if the request sounded like a go-ahead.

## Issue Template

```markdown
## Summary
[One-line description]

## Environment
- **Repo/Component**:
- **OS/Version**: (if relevant)

## Reproduction Steps
1. ...

## Expected Behavior
...

## Actual Behavior
...

## Error Details
```
[error text/stack trace, if applicable]
```

## Impact
[Critical/High/Medium/Low + why]

## Additional Context
...
```

Drop sections that don't apply — a one-line typo fix doesn't need "Reproduction Steps," and a feature request doesn't need "Actual Behavior."

## Severity Guide

- **Critical** — service/app down, data loss, security issue
- **High** — major feature broken, no workaround
- **Medium** — feature impaired, workaround exists
- **Low** — minor or cosmetic

## Guidelines

- **Be crisp.** No filler — every line should add information a triager needs.
- **Redact sensitive data.** Swap real secrets, tokens, connection strings, customer names, or internal-only URLs for placeholders like `[REDACTED]` before the issue goes anywhere public.
- **Images/GIFs**: reference inline as `![description](attachment-name.png)`. If the user has a local screenshot, they attach it via the GitHub web UI after creation (`gh issue create` doesn't upload images) — say so rather than silently dropping the reference.
- **If the target repo isn't obvious** from the working directory or conversation, ask which repo before drafting — don't guess at `owner/repo`.
