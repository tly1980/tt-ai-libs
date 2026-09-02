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

## Setup

1. Create the repo folder and clone into `main/`:
   ```
   git clone git@github.com:yourorg/<repo>.git workspaces/<repo>/main
   ```
2. From `main/`, add a worktree for the new branch:
   ```
   git worktree add ../<branch> -b <branch>
   ```
3. Inside the new worktree: copy `.env` from `main/` if one exists, run the project's dependency install (e.g. `uv sync`), and assign a unique port before starting any dev server.

## Cleanup (after a branch is merged)

1. In `main/`, switch to the default branch, `git pull`, then `git fetch --prune`.
2. Find local branches whose upstream is gone: `git branch -vv | grep ': gone]'`.
3. For each: `git worktree remove ../<branch> --force`, then `git branch -D <branch>` (force delete — see gotcha below on squash merges).
4. Run `git worktree prune` to clear any stale registrations.

---

## Lifecycle

```
Manager creates worktree
      → dispatches worker into it
Worker codes, commits, pushes, reports "done", exits
      → Manager opens PR, reviews diff + CI
      → Rework needed? Respawn a new worker into the SAME worktree with feedback
      → Approved? Merge (human-gated recommended initially)
Manager runs cleanup once merge is confirmed
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
