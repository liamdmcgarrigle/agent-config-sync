# agent-config-sync

An agent skill that sets up cross-tool configuration syncing so that **Claude Code, OpenCode, Codex, and Cursor** all share identical skills and MCP servers.

Claude Code acts as the single source of truth. Changes propagate to all other tools automatically via [vsync](https://github.com/nicepkg/vsync) and git hooks.

## Install

```bash
npx skills add liamdmcgarrigle/agent-config-sync --copy
```

Or install to a specific agent:

```bash
npx skills add liamdmcgarrigle/agent-config-sync -a claude-code --copy
```

> **Why `--copy`?** This skill uses [vsync](https://github.com/nicepkg/vsync) to sync skills across tools. vsync cannot follow symlinks, so skills must be copied as real files. Without `--copy`, the skills CLI creates symlinks that vsync silently skips.

<details>
<summary>Alternative: git submodule</summary>

For version-pinned installs tracked in your repo:

```bash
git submodule add https://github.com/liamdmcgarrigle/agent-config-sync.git .claude/skills/agent-config-sync
```

</details>

## What it does

Once installed, ask your agent to "set up agent sync" and it will:

1. **Configure vsync** — creates `.vsync.json` to sync skills, MCP servers, agents, and commands from Claude Code to Cursor, OpenCode, and Codex
2. **Set up git hooks** — a post-commit hook that auto-syncs after every commit, and a pre-commit hook that warns if Claude Code-only plugins are detected
3. **Search & install skills** — uses `npx skills` (from [skills.sh](https://skills.sh)) to find and install skills to Claude Code, then syncs to all tools
4. **Search & add MCP servers** — adds servers to `.mcp.json` and syncs with automatic format conversion (JSON/TOML/JSONC) per tool
5. **Manage skill submodules** — for version-pinned, team-shared skills via git submodules
6. **Update everything** — one workflow to update all skills, submodules, and re-sync

## Workflows

| Workflow | What it does |
|----------|-------------|
| `setup-agent-sync` | One-time project setup (vsync, git hooks, initial sync) |
| `find-skill` | Search skills.sh, install to Claude Code, sync to all tools |
| `add-skill-submodule` | Add a skill repo as a version-pinned git submodule |
| `find-mcp` | Search MCP registries, add server, sync to all tools |
| `update-agent-tools` | Update all skills + submodules + re-sync |

## How it works

```
Claude Code (.claude/)          -- SOURCE OF TRUTH
    |
    | npx @nicepkg/vsync sync
    |
    +---> Cursor (.cursor/)
    +---> OpenCode (.opencode/)
    +---> Codex (.codex/)
```

Skills use the universal [Agent Skills](https://agentskills.io) SKILL.md format, which works across all four tools. MCP server configs are converted between formats automatically by vsync.

## Requirements

- [Node.js](https://nodejs.org) (for npx)
- A git repository

vsync and npx skills are run via npx — no global installs needed.

## License

MIT
