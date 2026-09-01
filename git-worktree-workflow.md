# Git Worktree Workflow for Parallel Coding Agents

## Directory Layout

```
workspaces/
  aaa/
    main/                    ## original clone, always tracks origin/main
    feat-login-fix/          ## worktree, own branch, own .venv/.env
    feat-api-refactor/       ## worktree, own branch, own .venv/.env
  bbb/
    main/
    feat-my-feature-1/
    feat-my-feature-2/
```

### Naming conventions
- Repo folder is just the repo name (`aaa`, `bbb`) — no prefix needed.
- Main clone is named `main/`, not the repo name again — keeps scripts repo-agnostic (`cd workspaces/$repo/main` always works).
- Worktree folder name **==** branch name — keeps `git worktree list` / `git branch` trivial to reconcile.

---

## Per-Worktree Isolation

| Resource | Rule |
|---|---|
| `.venv` | Own venv per worktree via `uv sync` — never shared, avoids cross-branch dependency contamination |
| `.env` | Copied in at creation time from `main/.env` |
| Port | Unique per worktree, tracked in a state file so agents don't collide on dev servers |
| Git objects | Shared automatically via `main/.git` — worktrees are cheap, no duplicated history |

---

## Agent Roles

- **Manager (long-lived)** — owns all repo-mutating git operations: `worktree add`, `worktree remove`, `branch -D`, merge, PR open/close. Single serialization point that prevents races between parallel agents.
- **Workers (short-lived)** — only operate *inside* their assigned worktree: code, commit, push, report done, exit. Never touch git plumbing outside their own folder.

---

## Setup Commands

```bash
# create the main clone
mkdir -p workspaces/aaa
cd workspaces/aaa
git clone git@github.com:yourorg/aaa.git main

# create a worktree for a new feature branch
cd main
git worktree add ../feat-login-fix -b feat-login-fix
```

## Helper Scripts

**`new-worktree.sh <repo> <branch>`**
```bash
#!/usr/bin/env bash
repo=$1
branch=$2
cd "workspaces/$repo/main" || exit 1
git worktree add "../$branch" -b "$branch"
cd "../$branch"
[ -f ../main/.env ] && cp ../main/.env .env
uv sync
echo "Worktree ready at workspaces/$repo/$branch — remember to set a unique PORT"
```

**`cleanup-merged.sh <repo>`**
```bash
#!/usr/bin/env bash
set -e
repo=$1
cd "workspaces/$repo/main"

git checkout main
git pull
git fetch --prune

gone=$(git branch -vv | grep ': gone]' | awk '{print $1}')

for branch in $gone; do
  wt="../$branch"
  [ -d "$wt" ] && git worktree remove "$wt" --force
  git branch -D "$branch"
done

git worktree prune
echo "Cleanup complete for $repo"
```

---

## Lifecycle

```
Manager creates worktree
      → dispatches worker into it
Worker codes, commits, pushes, reports "done", exits
      → Manager opens PR, reviews diff + CI
      → Rework needed? Respawn a new worker into the SAME worktree with feedback
      → Approved? Merge (human-gated recommended initially)
Manager runs cleanup-merged.sh once merge is confirmed
      → worktree removed, branch deleted, state file updated
```

### Core principle
One actor (the **manager**) owns worktree/branch lifecycle. Everyone else (**workers**) operates inside an already-isolated sandbox and never needs to coordinate directly with siblings.

---

## Key Gotchas

- `git fetch --prune` only cleans up **remote-tracking refs**, not worktrees or local branches.
- `git worktree prune` only removes orphaned **registrations** (e.g. after a manual `rm -rf`) — it does not remove worktrees whose branch has merged.
- Squash-merged branches need `git branch -D` (force) — `-d` will refuse since git can't see the ancestry as merged.
- Always run `git worktree remove` instead of deleting folders by hand, or you'll leave stale registrations behind.
