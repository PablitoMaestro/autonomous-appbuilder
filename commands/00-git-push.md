Stage all changes, commit, and push to the current branch.

1. Run `git status` and `git diff` in parallel to understand what changed.
2. Run `git branch --show-current` to get the current branch name.
3. Draft a concise commit message based on the actual changes (not a generic one).
4. Stage all modified and untracked files — but skip any files that likely contain secrets (`.env`, credentials, keys) and warn the user if any are found.
5. Commit with the drafted message, appending the co-author trailer:
   `Co-Authored-By: Claude <noreply@anthropic.com>`
   (Optionally suffix with the actual model name if you know it from your runtime — e.g., `Claude Opus 4.7`. Don't guess; if unsure use plain `Claude`.)
6. Push to `origin <current-branch>`.
7. Confirm success and show the commit hash.
