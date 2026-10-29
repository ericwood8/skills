# Proving the check itself

Read this when: a check passed and you must show it can also fail, or you refactor with no behavior change, or two tables differ only in case.

- **Prove a check can fail:** after it passes, switch the fix off in the code under test (`if (false && ...)`), rebuild, run the check, look for the failure you expect, restore from a copy taken first and rebuild. Say in the notes that this was done.
- **Refactor with no behavior change:** write checks for what the script does not cover first (roles, for example), then run the same script on the old and new code: `git worktree add <scratch>/oldrepo HEAD`, build it, run with the worktree as the server repo; the old code must pass; then run the new code (same count, all pass); `git worktree remove --force` and `git worktree prune`. Check roles with real sign-ins (admin creates a manager, a report and an unrelated employee; assert 403/200 per route; capture `Location` in the script's call helper).
- **Case sensitivity:** test with two tables that differ only in case (`Movies`, `movies`) and write outputs to **distinct folders** (a Windows folder is case-insensitive, so one run overwrites the other).
