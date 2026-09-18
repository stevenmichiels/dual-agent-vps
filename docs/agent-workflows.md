# Agent Workflows

Claude Code CLI and Codex CLI are interactive coding tools for Git repositories
and disposable workspaces. They should not write directly to production runtime
directories, production env files, or production databases. Move changes to
live systems through reviewed Git diffs, CI/deploy scripts, Ansible, or an
explicit staging promotion.

## Dual-Agent Review Pattern

Keep one agent as the implementer and the other as a read-only reviewer. Do not
let both agents edit the same worktree at the same time.

The base role installs two reciprocal helpers:

- `codex-claude-review` lets Codex ask Claude Code to review the current diff.
- `claude-codex-review` lets Claude ask Codex CLI to review through Codex's
  native `codex review` command.

Both helpers can include recent plan context and write a Markdown report. They
do not give the reviewer edit tools, stage files, or create commits.

## Ask Claude to Review Codex Work

```sh
codex-claude-review \
  "Review the current diff as a strict senior engineer."

codex-claude-review -o claude-review.md \
  "Check whether this is overengineered."
```

The helper reviews `git diff HEAD` plus untracked files, uses plan-only Claude
permissions, and disables session persistence.

## Ask Codex to Review Claude Work

```sh
claude-codex-review \
  "Review the current diff as a strict senior engineer."

claude-codex-review --commit HEAD -o codex-review.md

claude-codex-review --base main \
  "Review this branch against main."
```

## Review Artifacts

Default reports go under local review directories. If a report should be
versioned, inspect it first and commit only that Markdown file in a separate
commit.

The helpers refuse likely secret-bearing paths and oversized diffs by default.
Override those protections only after manually reviewing what will be sent to
the second model. Plan context can also contain sensitive material and should
be disabled when it is irrelevant.

Source-controlled templates live in:

- `templates/codex-skills/claude-review/`
- `templates/claude-skills/codex-review/`
