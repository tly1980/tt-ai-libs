# Sub-agents

Claude Code subagent definitions derived from the rules in this repo. Each file
is a Markdown prompt with YAML frontmatter (`name`, `description`, `model`, `tools`).

| Agent | Source rule | Purpose |
|---|---|---|
| [worktree-manager.md](worktree-manager.md) | [../git-worktree-workflow.md](../git-worktree-workflow.md) | Owns worktree/branch lifecycle (create, isolate, clean up). Never writes feature code. |

## Install

Copy (or symlink) into your agents directory:

```bash
# user-wide
ln -sf /shared_rules/tt-ai-libs/sub_agents/worktree-manager.md ~/.claude/agents/

# or per-project
ln -sf /shared_rules/tt-ai-libs/sub_agents/worktree-manager.md <repo>/.claude/agents/
```

Then invoke with the Agent tool (`subagent_type: worktree-manager`) or by asking
Claude Code to "use the worktree-manager agent".
