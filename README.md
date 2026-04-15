# rebase-stack

A [Claude Code](https://docs.anthropic.com/en/docs/claude-code) skill that rebases an entire stack of PRs against the latest base branch, cascading changes through every branch using `git rebase --onto`.

## The problem

Stacked PRs are great for breaking large changes into reviewable pieces. But when review feedback lands in the middle of a stack, you need to rebase every branch above it — and ideally rebase the whole stack against the latest `main` while you're at it.

This is error-prone:
- Agents and scripts often use `git rebase --skip` instead of `--onto`, **silently dropping commits**
- Partial rebases leave the bottom of the stack stale against `main`
- Manually tracking old parent SHAs across 10+ branches is tedious and easy to get wrong

`/rebase-stack` handles the entire cascade correctly, every time.

## Install

```sh
npx skills add Iron-Ham/rebase-stack
```

### Manual install

Clone the repo and symlink it into your skills directory:

```sh
git clone git@github.com:Iron-Ham/rebase-stack.git ~/.claude/skills/rebase-stack
```

Or if you keep skills elsewhere:

```sh
git clone git@github.com:Iron-Ham/rebase-stack.git ~/path/to/rebase-stack
ln -s ~/path/to/rebase-stack ~/.claude/skills/rebase-stack
```

## Prerequisites

- [GitHub CLI](https://cli.github.com/) (`gh`) — used to discover the stack topology from open PRs
- Authenticated with `gh auth login`

## Usage

```
/rebase-stack                              # discover stack from current branch
/rebase-stack 42                           # stack containing PR #42
/rebase-stack Iron-Ham/auth-2-routes       # stack containing this branch
/rebase-stack --base develop               # rebase onto develop instead of main
/rebase-stack --dry-run                    # show the plan without executing
```

## What it does

1. **Discovers the stack** from GitHub PR metadata — walks base/head relationships to find the full ordered chain
2. **Records all branch SHAs** before touching anything
3. **Shows a rebase plan** and waits for your approval
4. **Rebases every branch** from the bottom of the stack to the top using `git rebase --onto`
5. **Handles conflicts** interactively — shows diffs, helps resolve, uses `--continue`
6. **Force-pushes** all branches with `--force-with-lease` (after confirmation)

## How it works

The key operation for each branch in the stack is:

```
git rebase --onto <new-parent> <old-parent-tip> <branch>
```

For single-commit-per-branch workflows, `<old-parent-tip>` is simply `branch~1` — the literal parent commit in the DAG. This is correct even when a parent branch was amended for review feedback, because the child's commit still points at the old parent commit.

For multi-commit branches, the skill falls back to `merge-base` with the remote's pre-amendment state.

### Why `--onto` and never `--skip`

- **`--onto`** reparents a branch: it takes commits from one base and replays them onto another. This is the correct operation for stack rebasing.
- **`--skip`** discards the current commit during a conflict. This **permanently loses work** and is never appropriate when rebasing a stack.

## License

MIT
