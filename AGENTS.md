<!-- zoridos-pr-finish v1 -->
## PR finish contract

Draft means still working. A finished PR left as draft is not done.
When work is finished, the worker MUST do these steps in order:

1. Commit with trailer `Agent: <agent> (<model>)`; push; verify that
   `git ls-remote origin <branch>` equals the full local HEAD SHA.
2. Put `Agent: <agent> (<model>)` on the PR body's first line (list every
   agent when several contributed), plus `Closes #<n>` (or `Refs #<n>` when deployment must follow) on its own line.
   Cross-repo `owner/repo#n` is allowed.
3. Run `gh pr ready <n>`.
4. Post a `SUMMARY:` comment naming the full head SHA, tests run and
   results, and limitations; include every contributing agent/model.

Workers never set or remove labels, merge, or review their own PR.
Human-authored PRs use `Agent: John (human)`.
If blocked, leave the PR draft and post `BLOCKED:` explaining why.
<!-- /zoridos-pr-finish -->
