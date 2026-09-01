# Claude Code → Cline CLI Migration Guide

> Date of migration: **2026-09-01**
> Environment: Windows 11, `D:\PVR` workspace, Cline CLI v3.0.60 (installed globally via `npm i -g cline`)
> Scope: project instructions (`CLAUDE.md` + sub-files), global skills (`~/.claude/skills`), MCP servers (`honeycomb`, `ollama`, Slack)

This document is the complete audit trail of migrating a Claude Code setup to Cline CLI. It explains the reasoning, records every command executed with its result, and lists what still needs manual follow-up.

---

## 1. Background research (what Cline actually reads)

Before touching anything, the following was confirmed against the official docs ([Cline Rules](https://docs.cline.bot/customization/cline-rules), [Cline Skills](https://docs.cline.bot/customization/skills), [Cline CLI overview](https://cline.bot/cli)):

| Claude Code concept | Cline CLI equivalent | Key finding |
|---|---|---|
| `CLAUDE.md` at project root | `AGENTS.md` at project root, or `.clinerules/` directory | **Cline does NOT read `CLAUDE.md`.** Auto-detected files: `AGENTS.md` (also `~/.agents/AGENTS.md` globally), `.clinerules/` (`.md`/`.txt`), `.cursorrules`, `.windsurfrules` |
| Global skills `~/.claude/skills/` | Global skills `~/.cline/skills/` | Cline reads **project-level** `.claude/skills/` for compatibility, but **NOT** the global `~/.claude/skills/`. Global skills must live in `~/.cline/skills/` |
| SKILL.md format | Same format | Both follow the Agent Skills standard (originated by Anthropic): directory + `SKILL.md` with YAML frontmatter `name` + `description`. `name` must exactly match the directory name (kebab-case, lowercase) |
| MCP servers in `~/.claude.json` | `cline mcp add` | Configs extracted from Claude Code and re-registered via CLI |
| MCP via plugin (`slack@claude-plugins-official`) | No equivalent plugin system | Must be re-added as a standalone MCP server in Cline |
| `~/.claude/settings.json` (permissions, hooks, auto-memory) | Auto-approve modes | No direct equivalent; Cline uses Plan/Act + auto-approve toggles instead of permission rules |

Other facts that shaped the plan:

- Skills use **progressive loading** in Cline: only metadata (~100 tokens) sits in context; full `SKILL.md` (<5k tokens) loads on trigger (auto-match by description, `use_skill` tool, or slash command like `/validate-mermaid`).
- Workspace rules take precedence over global rules on conflict.
- If a global and project skill share a name, **the global one wins** — keep that in mind if a BC repo later adds its own `.claude/skills/`.
- Cline global rules on Windows live in `Documents\Cline\Rules` (not yet used in this migration).

---

## 2. Inventory of the source setup

**Instruction files at `D:\PVR`:**

- `CLAUDE.md` — master instructions (mirrored to `vaibhavPH/claude-pvr` on GitHub)
- `LINEARIS.md`, `MERMAID.md`, `DEPLOYMENT.md`, `SLACK.md`, `onboarding.md` — sub-files referenced by path from `CLAUDE.md`
- **No conversion needed for sub-files** — both agents read them on demand by path; only the entry file must exist in a format Cline detects.

**Skills in `~/.claude/skills/` (12 directories + 2 helper `.bat` files):**

`forticlient-check-connection-status`, `local-diff-walk`, `meeting-notes`, `postgres-get`, `pr-changelog-walk`, `purge-claude-sessions`, `rabbitmq-connect`, `rabbitmq-publish`, `review-sequential`, `slack-post`, `validate-mermaid`, `weekly-code-docs`

The two `.bat` files (`commit_claude_skills_to_vibsworld.bat`, `update-apps.bat`) are not skills and were intentionally **not** copied.

**MCP servers (found in top-level `mcpServers` of `~/.claude.json`; no project-level `.mcp.json` exists):**

```json
{
  "ollama": {
    "type": "stdio",
    "command": "npx",
    "args": ["-y", "ask-ollama-mcp"],
    "env": { "OLLAMA_HOST": "http://127.0.0.1:11434" }
  },
  "honeycomb": {
    "type": "http",
    "url": "https://mcp.honeycomb.io/mcp"
  }
}
```

Slack MCP is provided by the Claude Code plugin `slack@claude-plugins-official` (enabled in `~/.claude/settings.json`), not by an `mcpServers` entry.

**Decision point raised to the user** — how Cline should receive project instructions. Options considered:

1. **Copy `CLAUDE.md` → `AGENTS.md`** (chosen) — `CLAUDE.md` stays the source of truth; re-copy after edits. `AGENTS.md` is also the cross-tool standard (Codex, Gemini CLI, etc. read it).
2. Rename `CLAUDE.md` → `AGENTS.md` — single source, but loses the Claude Code entry point.
3. Split into `.clinerules/` topic files — cleanest organization (per-file toggling) but the most work and the highest drift risk.

---

## 3. Migration steps (audit trail)

### Step 1 — Copy instructions: `CLAUDE.md` → `AGENTS.md`

```bash
cp D:/PVR/CLAUDE.md D:/PVR/AGENTS.md
```

A header comment was prepended so future readers know not to edit the copy directly:

```html
<!-- GENERATED COPY — source of truth is CLAUDE.md in this directory. Edit CLAUDE.md and re-copy:
`cp D:/PVR/CLAUDE.md D:/PVR/AGENTS.md` (then re-add this header).
Cline CLI reads AGENTS.md, not CLAUDE.md. -->
```

**Sync habit going forward:** after every edit to `CLAUDE.md`, re-run the copy command and re-add the header. The existing GitHub-mirror flow (`vaibhavPH/claude-pvr`) is unchanged — it mirrors `CLAUDE.md`, which remains the source of truth.

### Step 2 — Copy skills to `~/.cline/skills/`

```bash
mkdir -p ~/.cline/skills
cp -r ~/.claude/skills/*/ ~/.cline/skills/
```

The `*/` glob copies only **directories**, which is why the two helper `.bat` files were automatically excluded. Result — all 12 skills now in `~/.cline/skills/`:

```
forticlient-check-connection-status/  meeting-notes/         rabbitmq-publish/    weekly-code-docs/
local-diff-walk/                     postgres-get/          review-sequential/
pr-changelog-walk/                   rabbitmq-connect/      slack-post/
purge-claude-sessions/               validate-mermaid/
```

Bundled scripts (`validate.mjs`, `render.mjs`, `node_modules/`, etc.) were carried over untouched — skills are self-contained and work identically.

### Step 3 — Verify skill frontmatter

Cline requires `name:` in the `SKILL.md` frontmatter to **exactly match the directory name**. Verified all 12:

```
forticlient-check-connection-status → name: forticlient-check-connection-status
local-diff-walk                     → name: local-diff-walk
meeting-notes                       → name: meeting-notes
postgres-get                        → name: postgres-get
pr-changelog-walk                  → name: pr-changelog-walk
purge-claude-sessions               → name: purge-claude-sessions
rabbitmq-connect                    → name: rabbitmq-connect
rabbitmq-publish                    → name: rabbitmq-publish
review-sequential                   → name: review-sequential
slack-post                          → name: slack-post
validate-mermaid                    → name: validate-mermaid
weekly-code-docs                   → name: weekly-code-docs
```

All 12 pass — no edits required.

### Step 4 — Register MCP server: honeycomb

```bash
cline mcp add honeycomb --transport http --yes --json https://mcp.honeycomb.io/mcp
```

Result:

```json
{"name":"honeycomb","status":"installed","transport":{"type":"streamableHttp","url":"https://mcp.honeycomb.io/mcp"},"warnings":[]}
```

Note: Cline normalized the transport to `streamableHttp` — equivalent to Claude Code's `"type": "http"`.

### Step 5 — Register MCP server: ollama

```bash
cline mcp add ollama --transport stdio --yes --json npx -- -y ask-ollama-mcp
```

Result:

```json
{"name":"ollama","status":"installed","transport":{"type":"stdio","command":"npx","args":["-y","ask-ollama-mcp"]},"warnings":[]}
```

The `OLLAMA_HOST=http://127.0.0.1:11434` env var from the Claude config was **dropped** — `127.0.0.1:11434` is Ollama's default host, so the variable was redundant, and the `cline mcp add` CLI has no `--env` flag anyway.

---

## 4. How to verify the migration

Run `cline` in `D:\PVR`:

1. **Rules** — the Rules panel (also reachable via `cline config` → Rules tab) should list `AGENTS.md`, toggleable on/off.
2. **Skills** — type `/` in chat; all 12 skills should appear as slash commands (`/validate-mermaid`, `/slack-post`, …). Skills must be enabled (toggled on) in the Skills menu.
3. **MCP** — `cline mcp list` (or the MCP panel) should show `honeycomb` and `ollama` as connected.

---

## 5. Remaining follow-ups

| Item | Status | Notes |
|---|---|---|
| Slack MCP | ⚠️ Not migrated | Came from Claude Code plugin `slack@claude-plugins-official`; plugins don't carry over. Add a Slack MCP server in Cline (`cline mcp add slack ...`) when needed. Much of `SLACK.md` uses direct `Invoke-RestMethod`, which works as-is. |
| `slack-post` skill review | ⚠️ Needed after Slack MCP is added | The skill references Claude's tool names (`mcp__plugin_slack_slack__*`); Cline's MCP tools are named differently. Update instructions once the Slack MCP server exists in Cline. |
| `purge-claude-sessions` skill | Kept, but Claude-specific | Harmless in Cline; consider skipping/ignoring. |
| Claude-specific instructions inside `AGENTS.md` | Acceptable noise | Plan mode, artifacts, memory directives, etc. are inert in Cline. Optionally prune later — but remember pruning would diverge the copy from `CLAUDE.md`. |
| Permissions / hooks / auto-memory | No equivalent | Cline uses Plan/Act mode (Tab) and auto-approve (Shift+Tab) instead of permission rules; no hooks migration. |
| Global rules | Not used yet | Windows location: `Documents\Cline\Rules` — available if cross-project personal preferences are wanted later. |
| Keeping skills in sync | Ongoing habit | New/updated skills must be copied to **both** `~/.claude/skills/` and `~/.cline/skills/` while both agents are in use. |

---

## 6. Quick reference — the maintenance commands

```bash
# Re-sync instructions after editing CLAUDE.md
cp D:/PVR/CLAUDE.md D:/PVR/AGENTS.md
# (then re-add the generated-copy header at the top of AGENTS.md)

# Copy a new/updated skill to Cline
cp -r ~/.claude/skills/<skill-name> ~/.cline/skills/

# Add an MCP server (remote)
cline mcp add <name> --transport http --yes <url>

# Add an MCP server (stdio) — args after the double dash
cline mcp add <name> --transport stdio --yes <command> -- <args...>

# List / remove MCP servers
cline mcp list
cline mcp remove <name>
```