---
name: agent-config-sync
license: MIT
description: >
  MUST be used whenever the user wants to add, install, find, or search for a skill,
  MCP server, or plugin. Also use when setting up cross-tool agent syncing across
  Claude Code, OpenCode, Codex, and Cursor. This skill ensures all tools stay
  identical by installing to Claude Code first and syncing via vsync. Use PROACTIVELY
  when the user says things like "add an MCP server", "find a skill for X",
  "install a plugin", "add a tool", "search for skills", "set up MCP", "I need a
  skill that does X", "add svelte MCP", or any request to add capabilities to the
  agent. Also triggers on: vsync, cross-tool parity, agent sync, evaluating AI tools,
  /setup-agent-sync, /find-skill, /add-skill-submodule, /find-mcp, /update-agent-tools.
---

# Agent Sync Setup

This skill sets up and manages a system where Claude Code, OpenCode, Codex, and Cursor
all have **identical** skills and MCP servers. Claude Code is the single source of truth,
and changes propagate to all other tools automatically via vsync and git hooks.

## Architecture Overview

```
Claude Code (.claude/)          -- SOURCE OF TRUTH
    |
    | npx @nicepkg/vsync sync
    |
    +---> Cursor (.cursor/)
    +---> OpenCode (.opencode/)
    +---> Codex (.codex/)
```

**What syncs:** Skills (SKILL.md files), MCP servers, agents, commands.
**What does NOT sync:** Plugins. Plugins are Claude Code-only and break cross-tool parity.
Use skills instead of plugins whenever possible.

**Key tools:**
- **vsync** (`npx @nicepkg/vsync`) - Syncs configs across tools, handles format conversion
- **npx skills** (from skills.sh / vercel-labs/skills) - Universal skill package manager
- **Git submodules** - For version-pinned, team-shared skill repos

**Skill directories per tool:**
| Tool | Skills Dir | MCP Config |
|------|-----------|------------|
| Claude Code | `.claude/skills/` | `.mcp.json` (JSON) |
| Cursor | `.cursor/skills/` | `mcp.json` (JSON) |
| OpenCode | `.opencode/skills/` | `opencode.json` (JSON, project root) |
| Codex | `.codex/skills/` | `config.toml` (TOML) |

vsync handles all format conversions automatically.

**Sync support per tool:**
| Feature | Claude Code | Cursor | OpenCode | Codex |
|---------|------------|--------|----------|-------|
| Skills | Yes (source) | Yes | Yes | Yes |
| MCP | Yes (source) | Yes | Yes | Yes |
| Agents | Yes (source) | No | Yes | No |
| Commands | Yes (source) | Yes | Yes | No |

---

## Workflow: setup-agent-sync

Run this once per project to set up the full sync system.

### Step 1: Create `.vsync.json`

Create this file at the project root:

```json
{
  "version": "1.0.0",
  "level": "project",
  "source_tool": "claude-code",
  "target_tools": [
    "codex",
    "cursor",
    "opencode"
  ],
  "sync_config": {
    "skills": true,
    "mcp": true,
    "agents": true,
    "commands": true
  },
  "use_symlinks_for_skills": false
}
```

Important: `use_symlinks_for_skills` must be `false` to avoid errors when no skills
directory exists yet. vsync will copy files instead of symlinking.

### Step 2: Create post-commit git hook

Create `.git/hooks/post-commit` with executable permissions (`chmod +x`):

```sh
#!/bin/sh

# Prevent infinite loop when this hook triggers its own commit
[ "$VSYNC_RUNNING" = "1" ] && exit 0

# Sync configs from Claude Code to all target tools
npx @nicepkg/vsync sync -y

# If vsync changed any config files, auto-commit them
SYNC_FILES=".mcp.json opencode.json AGENTS.md CLAUDE.md .cursor .opencode .codex"
if ! git diff --quiet $SYNC_FILES 2>/dev/null; then
  git add $SYNC_FILES 2>/dev/null
  VSYNC_RUNNING=1 git commit --no-verify \
    --author="vsync agent sync <vsync@local>" \
    -m "chore: sync agent configs [vsync]"
fi
```

### Step 3: Create pre-commit git hook

Create `.git/hooks/pre-commit` with executable permissions (`chmod +x`):

```sh
#!/bin/sh

# Warn if plugins are being used instead of skills
# Plugins are Claude Code-only and break cross-tool parity
if [ -f ".claude/settings.json" ]; then
  if grep -q '"enabledPlugins"' .claude/settings.json 2>/dev/null; then
    echo ""
    echo "============================================================"
    echo "  WARNING: Claude Code plugins detected in settings.json"
    echo "============================================================"
    echo ""
    echo "  Plugins only work in Claude Code and break cross-tool"
    echo "  parity with OpenCode, Codex, and Cursor."
    echo ""
    echo "  For cross-tool compatibility, prefer:"
    echo "    npx skills find \"<keyword>\"     # search for a skill"
    echo "    npx skills add <repo> -a claude-code  # install skill"
    echo ""
    echo "  Skills use the universal SKILL.md format and sync to"
    echo "  all tools via vsync automatically."
    echo "============================================================"
    echo ""
  fi
fi
```

This hook warns but does NOT block the commit. Some plugins (like typescript-lsp)
are legitimately Claude Code-only and that is fine -- the warning just reminds the
user to prefer skills when a cross-tool alternative exists.

### Step 4: Run initial sync

```bash
npx @nicepkg/vsync sync -y
```

### Step 5: Verify

Check that target tool directories were created:
```bash
ls -la .cursor/  .opencode/  .codex/  2>/dev/null
```

---

## Workflow: find-skill

Search for and install a skill from the skills.sh registry.

### Step 1: Search

```bash
npx skills find "<keyword>"
```

This searches skills.sh, the largest cross-tool skill directory. Results show skill
name, description, and install count.

### Step 2: Install to Claude Code only

```bash
npx skills add <owner/repo> --skill <skill-name> -a claude-code
```

Always install to `claude-code` only. Claude Code is the source of truth -- vsync
handles distribution to the other tools.

Do NOT use `-a cursor -a opencode -a codex`. Let vsync do its job.

### Step 3: Sync to all tools

```bash
npx @nicepkg/vsync sync -y
```

Or just commit -- the post-commit hook runs vsync automatically.

### Example workflow

```bash
# User wants a Svelte skill
npx skills find "svelte"

# Install the one they pick
npx skills add spences10/svelte-claude-skills --skill svelte5-runes -a claude-code

# Sync (or just commit and the hook does it)
npx @nicepkg/vsync sync -y
```

---

## Workflow: add-skill-submodule

Add a skill repo as a git submodule for version-pinned, team-shared skills.
Use this instead of `npx skills add` when:
- The team needs everyone on the exact same version
- The skill repo is private
- You want the skill tracked in your repo's git history

### Step 1: Add the submodule

```bash
git submodule add <repo-url> .claude/skills/<name>
```

The repo must contain one or more directories with `SKILL.md` files in the
universal Agent Skills format:

```
<name>/
  skill-one/
    SKILL.md
  skill-two/
    SKILL.md
```

Or if the repo IS a single skill:

```
<name>/
  SKILL.md
```

### Step 2: Commit the submodule

```bash
git add .gitmodules .claude/skills/<name>
git commit -m "feat: add <name> skill submodule"
```

The post-commit hook will automatically run vsync to sync the skill to all target tools.

### Step 3: Team members initialize submodules

After cloning, team members need to run:

```bash
git submodule update --init --recursive
```

### Updating a submodule to latest

```bash
git submodule update --remote --merge .claude/skills/<name>
git add .claude/skills/<name>
git commit -m "chore: update <name> skill submodule"
```

---

## Workflow: find-mcp

Search for and add an MCP server. MCP servers are runtime services (npm packages,
HTTP endpoints) -- they go in `.mcp.json`, NOT as git submodules.

### Step 1: Search

MCP servers can be discovered on these registries:
- **smithery.ai** (7,300+ servers) - https://smithery.ai
- **mcp.so** (17,900+ servers) - https://mcp.so
- **glama.ai** (largest collection) - https://glama.ai/mcp/servers
- **pulsemcp.com** (8,600+ servers) - https://pulsemcp.com/servers

Help the user search for what they need. If they describe a use case, suggest
relevant MCP servers from these registries.

### Step 2: Add to Claude Code

For npm-based servers:
```bash
claude mcp add <server-name> -- npx -y @package/mcp-server
```

For HTTP-based servers, edit `.mcp.json` directly:
```json
{
  "mcpServers": {
    "server-name": {
      "type": "http",
      "url": "https://example.com/mcp"
    }
  }
}
```

For servers with environment variables:
```json
{
  "mcpServers": {
    "server-name": {
      "command": "npx",
      "args": ["-y", "@package/mcp-server"],
      "env": {
        "API_KEY": "${API_KEY}"
      }
    }
  }
}
```

### Step 3: Sync to all tools

```bash
npx @nicepkg/vsync sync -y
```

vsync reads `.mcp.json` and converts it to each tool's native format:
- Cursor: `.cursor/mcp.json` (JSON, `mcpServers` key)
- OpenCode: `opencode.json` at project root (JSON, `mcp` key, `type: "remote"` for HTTP)
- Codex: `.codex/config.toml` (TOML, `mcp_servers` key)

vsync also converts environment variable syntax between tools
(`${VAR}` vs `${env:VAR}` vs `{env:VAR}`).

### Step 4: Commit

```bash
git add .mcp.json
git commit -m "feat: add <server-name> MCP server"
```

The `.mcp.json` file should be committed to version control so team members
get the same MCP server configuration.

---

## Workflow: update-agent-tools

Update all skills and MCP configs across all tools.

```bash
# Update skills installed via npx skills
npx skills update

# Update skills installed as git submodules
git submodule update --remote --merge

# Re-sync everything to all tools
npx @nicepkg/vsync sync -y
```

Run this periodically or whenever you want the latest versions of your skills.

If submodules were updated, commit the changes:
```bash
git add .claude/skills/
git commit -m "chore: update skill submodules"
```

The post-commit hook will handle syncing automatically.

---

## Troubleshooting

### vsync says "everything is up to date" but target tools are empty
- Check that `.claude/skills/` actually contains skill directories with SKILL.md files
- vsync syncs what exists in the source -- if the source is empty, targets will be too
- Run `ls -la .claude/skills/` to verify

### vsync fails with "Source skills directory does not exist"
- Set `"use_symlinks_for_skills": false` in `.vsync.json`
- This happens when symlink mode is enabled but no skills directory exists yet

### Pre-commit hook warns about plugins
- This is expected if you have Claude Code plugins installed
- For cross-tool parity, prefer `npx skills add` over `/plugin install`
- Some plugins (like LSP servers) are legitimately Claude Code-only -- the warning
  is informational, not blocking

### Skills not appearing in target tools after sync
- Verify the skill has a valid `SKILL.md` with YAML frontmatter (`name` and `description` required)
- Run `npx @nicepkg/vsync sync -y` manually to see any errors
- Check target directories: `.cursor/skills/`, `.opencode/skills/`, `.codex/skills/`

### Git submodule not initialized for team members
- Team members must run `git submodule update --init --recursive` after cloning
- Add this to your README or onboarding docs
