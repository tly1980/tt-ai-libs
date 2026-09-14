# Sub-agents

Claude Code subagent definitions derived from the rules in this repo. Each file
is a Markdown prompt with YAML frontmatter (`name`, `description`, `model`, `tools`).

| Agent | Source rule | Purpose |
|---|---|---|
| [worktree-manager.md](worktree-manager.md) | [../git-worktree-workflow.md](../git-worktree-workflow.md) | Owns worktree/branch lifecycle (create, isolate, clean up). Never writes feature code. |

## Scripts

Agents delegate mechanics to scripts in [scripts/](scripts/) so a task is one
command instead of a dozen reasoning steps — cheaper in tokens and less error-prone.

| Script | Used by | Does |
|---|---|---|
| [scripts/wt](scripts/wt) | worktree-manager | `create` / `cleanup` / `list` / `status` for the `<repo>/main` + sibling-worktree layout. Copies `.env`, assigns a unique `PORT` (tracked in `<repo>/.worktree-ports.json`), runs `uv sync` / `npm ci` / `cargo fetch`, refuses to remove dirty or unmerged worktrees unless `--yes`. Requires `git`, `jq`; uses `gh` when available to confirm a PR was merged. |

Scripts are usable by humans too: `scripts/wt` with no args prints usage.

## Install

Copy (or symlink) into your agents directory:

```bash
# user-wide
ln -sf /shared_rules/tt-ai-libs/sub_agents/worktree-manager.md ~/.claude/agents/

# or per-project
ln -sf /shared_rules/tt-ai-libs/sub_agents/worktree-manager.md <repo>/.claude/agents/
```

Prompts reference scripts by absolute path under `/shared_rules/tt-ai-libs`; adjust
if this repo is checked out elsewhere.

Then invoke with the Agent tool (`subagent_type: worktree-manager`) or by asking
Claude Code to "use the worktree-manager agent".
