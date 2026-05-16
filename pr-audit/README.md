# pr-audit

Automated pull request review for Claude Code.

You point it at your changes; it commits them, opens a PR on GitHub, reviews
the code, and posts its findings as comments — the same way a senior engineer
would review your PR, but in a couple of minutes.

## Install

In Claude Code, run these **two commands one at a time** — wait for the first
to finish before running the second.

1. Add the marketplace:

   ```
   /plugin marketplace add https://github.com/Crscristi28/pr-audit-plugin
   ```

2. Install the plugin:

   ```
   /plugin install pr-audit@pr-audit-plugin
   ```

## Usage

Run inside any git repository:

```
/pr-audit          # review the changes you have not committed yet
/pr-audit 247      # review an existing PR, here PR #247
```

**`/pr-audit` with no number** takes your uncommitted work, creates a branch,
commits it, pushes, opens a PR, and reviews it — all in one step.

**`/pr-audit <number>`** skips the git steps and just reviews a PR that already
exists.

## What it checks

The plugin reads your diff and looks for real problems, including:

- **Security** — SQL injection, hardcoded secrets, missing authorization,
  unsafe input handling.
- **Correctness** — logic bugs, null/undefined crashes, missing error handling,
  race conditions.
- **Performance** — N+1 database queries, unbounded loops, blocking calls.
- **Accessibility** — missing labels, keyboard traps, unlabeled images (for UI
  changes).
- **Architecture, API contracts, test coverage** — for larger or higher-risk
  changes.

Small, low-risk changes get a quick single-pass review. Large or sensitive
changes (touching auth, database schema, or public APIs) get the full
treatment, with each area reviewed by its own dedicated pass running in
parallel.

## What you get

- **Inline comments** on the GitHub PR, one per issue, anchored to the exact
  line. Where the fix is simple, the comment includes a one-click
  "Commit suggestion" button.
- **A summary comment** with the overall verdict and issue counts.
- **A short brief in the chat** — the verdict, the top concerns, and the PR
  link.

After the review, the plugin asks if you want it to apply the fixes right away.
You can say yes and it fixes them in the same session, or say no and fix them
yourself from the PR comments.

## Tip: review in a fresh session

If Claude wrote the code, run `/pr-audit` in a **new** Claude Code session
rather than the one that built the feature. A fresh session has no attachment
to the code and reviews it more honestly. Same project, new chat. Optional, but
it catches more.

## Custom rules (optional)

The plugin already knows the common bugs. But it cannot know **your team's own
rules** — things like "every payment must be written to the audit log" or "API
routes must use our `requireAuth` middleware". Those are specific to your
project.

To have the plugin check them too, create a file at `.pr-audit/rules.md` in
your repository root and write your rules under a heading:

```markdown
## Security

- All API routes must use the `requireAuth` middleware.

## Business Logic

- Every database write must run inside a transaction.
```

The plugin reads this file automatically on every run. If it is not there,
nothing changes — the built-in checks still run.

## Requirements

- Claude Code (Pro, Max, or API)
- `gh` — the GitHub CLI, signed in (`gh auth login`)
- `git` with permission to push to the repository

## What it will not do

- It never merges your PR — you merge it yourself once you are happy.
- It never approves a PR on your behalf.

## License

MIT
