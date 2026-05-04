Stage all changes, commit, and push to the current branch.

1. Run `git status` and `git diff` in parallel to understand what changed.
2. Run `git branch --show-current` to get the current branch name.
3. Draft a concise commit message based on the actual changes (not a generic one).
4. Stage all modified and untracked files — but skip any files that likely contain secrets (`.env`, credentials, keys) and warn the user if any are found.
5. Commit with the drafted message. Do NOT hand-write a co-author trailer — your AI tool (Claude Code, Codex, etc.) injects the correct attribution itself based on its `attribution.commit` setting (or equivalent). Hand-writing it leads to mismatches when the same workflow runs on different tools.
6. Push to `origin <current-branch>`.
7. Confirm success and show the commit hash.
