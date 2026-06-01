# Claude Code: Beginner to Ninja — Step-by-Step Guide

A structured learning path that takes you from zero to expert with Claude Code CLI. Follow the levels in order — each one builds on the previous.

---

## Level 1 — Beginner: Get Claude Code Running

**Goal:** Install Claude Code, understand what it is, and complete your first session.

### 1.1 Install and Authenticate

```bash
# Install via npm (requires Node.js 18+)
npm install -g @anthropic-ai/claude-code

# Launch in any git repo
cd your-project
claude
```

Sign in with your Anthropic account when prompted. Claude Code works inside your terminal — it can read files, edit code, run commands, and use git, all with your approval.

### 1.2 Core Concepts in 5 Minutes

| Concept | What It Means |
|---------|---------------|
| **Session** | One conversation in your terminal. Starts with `claude`, ends when you type `/exit` |
| **Turn** | One message from you → one response from Claude (which may include file edits, commands, etc.) |
| **Context window** | How much conversation Claude can remember. Old messages get summarized as the session grows |
| **Permission mode** | How much autonomy Claude has — `default` (asks first), `auto-accept` (less prompting), or `plan` (discuss before acting) |
| **Tool use** | Claude doesn't just chat — it calls tools (Read, Edit, Bash, Grep, Glob, etc.) to act on your codebase |

### 1.3 Your First Session Checklist

- [ ] Run `claude` inside a git repo
- [ ] Ask Claude to explain a file: *"Walk me through what `src/main.ts` does"*
- [ ] Ask Claude to find something: *"Where is the database connection configured?"*
- [ ] Ask Claude to make a small change: *"Add a health-check endpoint to the API"*
- [ ] Review the diff Claude proposes, then approve or reject
- [ ] Type `/exit` to end the session

> **Repo to explore:** [Claude-Code-Templates](https://github.com/davila7/claude-code-templates) — grab a starter config so you don't start from zero.

---

## Level 2 — Intermediate: Work Smarter, Not Harder

**Goal:** Use CLAUDE.md, slash commands, and hooks to reduce repetitive prompting.

### 2.1 CLAUDE.md — Your Project's Brain

Claude Code reads `CLAUDE.md` at the root of every project on launch. Think of it as a persistent `.cursorrules` or `.editorconfig` for AI.

```markdown
# CLAUDE.md

## Project
- .NET 8 microservices architecture
- EventStoreDB for event sourcing, Postgres for read models

## Conventions
- Use file-scoped namespaces
- Prefer record types for DTOs
- All API endpoints return `Result<T>` not exceptions

## Rules
- Never commit directly to main — always branch
- Run `bandar unit-tests` before asking for review
```

Put project conventions, tech stack, coding standards, and workflow rules here. Claude follows these every session without you re-explaining.

### 2.2 Slash Commands and Skills

| Command | What It Does |
|---------|-------------|
| `/help` | List all commands and shortcuts |
| `/clear` | Wipe the current context (fresh start) |
| `/compact` | Summarize the conversation so far (frees context window) |
| `/cost` | Show token usage and spend for the session |
| `/memory` | View or edit persistent memories across sessions |
| `/init` | Generate a CLAUDE.md for the current project |

Skills are custom slash commands you define in `.claude/skills/`. A skill is just a markdown file with instructions Claude follows when invoked.

### 2.3 Hooks — Automate the Boring Stuff

Hooks let you run scripts before or after Claude's actions. Define them in `.claude/settings.json`:

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "command": "echo 'Running a shell command...'"
      }
    ],
    "PostToolUse": [
      {
        "matcher": "Write|Edit",
        "command": "npx prettier --write $CLAUDE_FILE_PATH"
      }
    ]
  }
}
```

Common hook patterns:
- **Auto-format** after every file edit (Prettier, dotnet format, etc.)
- **Run linting** before accepting code changes
- **Log commands** for audit trails

> **Repo to explore:** [Claude-Code-Best-Practice](https://github.com/shanraisshan/claude-code-best-practice) — real-world patterns for CLAUDE.md, hooks, and commands.

---

## Level 3 — Proficient: Agents, MCP, and Multi-Step Workflows

**Goal:** Orchestrate Claude Code for complex, multi-step tasks without babysitting it.

### 3.1 Agent Loops and Subagents

Claude Code can spawn subagents — independent Claude instances that each focus on a specific task. This is how it handles large changes:

1. **Orchestrator** breaks the task into subtasks
2. Each **subagent** works on one subtask (reading, editing, testing)
3. The orchestrator collects results and moves to the next step

You see this in action when Claude says *"I'll use a subagent to handle this."* You can also define custom subagent roles in `.claude/agents/`.

### 3.2 MCP (Model Context Protocol) Servers

MCP servers give Claude Code access to external tools and data. Connect them in `.claude/settings.json`:

```json
{
  "mcpServers": {
    "github": {
      "command": "docker",
      "args": ["run", "-i", "--rm", "ghcr.io/github/github-mcp-server"]
    },
    "postgres": {
      "command": "npx",
      "args": ["-y", "@anthropic-ai/mcp-server-postgres", "postgresql://localhost/mydb"]
    }
  }
}
```

Popular MCP servers:
- **GitHub** — PRs, issues, code search
- **Postgres / MySQL** — query databases directly
- **Filesystem** — sandboxed file access
- **Puppeteer** — browser automation for testing

Once connected, Claude can use these tools natively — no copy-pasting data into the chat.

### 3.3 The Plan → Execute → Verify Pattern

For non-trivial tasks, switch Claude to **plan mode** (`/plan` or just describe the goal and say "plan first"):

1. **Plan** — Claude outlines the approach, you approve
2. **Execute** — Claude makes the changes step by step
3. **Verify** — Claude runs tests, checks the diff, and confirms everything works

This prevents Claude from going off the rails on large refactors.

> **Repo to explore:** [Get-Shit-Done](https://github.com/gsd-build/get-shit-done) — a structured workflow skill that enforces discussion → planning → execution → verification → shipping.

---

## Level 4 — Advanced: Custom Skills, Memory, and Project Automation

**Goal:** Build your own skills and configure Claude Code to match your team's workflow.

### 4.1 Building Custom Skills

A skill is a markdown file in `.claude/skills/` that Claude treats as an instruction set when invoked:

```markdown
<!-- .claude/skills/review.md -->
# Code Review Skill

When I type /review, perform a thorough code review:

1. Check for bugs, edge cases, and error handling
2. Verify naming conventions match the project style
3. Look for performance issues (N+1 queries, unnecessary allocations)
4. Ensure tests cover the changed behavior
5. Summarize findings in a structured table

Format: | File | Line | Severity | Finding | Suggestion |
```

Skills can reference each other, accept arguments, and chain into multi-step workflows.

### 4.2 Memory — Teach Claude Once, Remember Forever

Claude Code has a persistent memory system at `~/.claude/projects/<project>/memory/`. Use `/memory` to save facts across sessions:

- *"Remember that this project uses MediatR for CQRS"*
- *"Our API prefix is always /api/v2"*
- *"Never use float — always decimal for currency"*

Claude reads these memories at the start of every session. It's like CLAUDE.md but for your personal preferences rather than project rules.

### 4.3 Project-Level Automation

| File | Purpose |
|------|---------|
| `.claude/settings.json` | Hooks, MCP servers, permissions |
| `.claude/skills/*.md` | Custom slash commands |
| `.claude/agents/*.md` | Custom subagent role definitions |
| `CLAUDE.md` | Project instructions (always loaded) |
| `~/.claude/CLAUDE.md` | Global instructions (all projects) |

A well-configured project means Claude already knows your conventions, has the right tools, and follows your rules — before you type a single prompt.

> **Repo to explore:** [Awesome-Claude-Code](https://github.com/hesreallyhim/awesome-claude-code) — the definitive directory of community skills, hooks, and plugins.

---

## Level 5 — Expert: Multi-Agent Orchestration

**Goal:** Run parallel agents on complex tasks and manage worktree isolation.

### 5.1 Worktree Isolation

When Claude spawns agents that modify code, each one gets its own git worktree — an isolated copy of the repo. This means:

- Agents don't step on each other's changes
- You can review each agent's work independently
- Failed experiments leave no trace in your working tree

### 5.2 Role-Based Agent Teams

You can define agents with specific roles in `.claude/agents/`:

```
.claude/agents/
  ceo.md         — breaks user stories into tasks, prioritizes
  designer.md    — focuses on UX and API design
  engineer.md    — implements features, writes tests
  reviewer.md    — code review, security audit
  qa.md          — writes and runs test plans
```

Each agent gets a focused system prompt. The orchestrator delegates to the right agent based on the task.

### 5.3 Parallel Execution

For large tasks (migrations, multi-file refactors), Claude can:

1. **Fan out** — spawn multiple agents for independent subtasks
2. **Pipeline** — chain agents sequentially (plan → implement → test)
3. **Verify** — spawn adversarial agents to review each other's work

This is where Claude Code stops being "a smarter autocomplete" and becomes a genuine development partner.

> **Repos to explore:**
> - [Claude Swarm](https://github.com/affaan-m/claude-swarm) — multi-agent orchestration with terminal UI
> - [GStack](https://github.com/garrytan/gstack) — role-based agent team (CEO, Designer, Eng Manager, QA)
> - [Awesome-Claude-Code-Subagents](https://github.com/VoltAgent/awesome-claude-code-subagents) — library of specialized subagent definitions

---

## Level 6 — Ninja: System Prompts, Custom Harnesses, and Deep Customization

**Goal:** Understand Claude Code's internals deeply enough to build your own tools on top of it.

### 6.1 Understanding System Prompts

Every Claude Code session starts with a system prompt that tells Claude how to behave — what tools it has, how to format responses, when to ask for permission. Studying these prompts reveals:

- How Claude decides which tool to use
- What instructions shape its coding style
- How the context window is managed
- What changes between versions

> **Repo to explore:** [Claude-Code-System-Prompts](https://github.com/Piebald-AI/claude-code-system-prompts) — tracks prompt changes across releases. Also see [System-Prompts-and-Models-of-AI-Tools](https://github.com/x1xhlol/system-prompts-and-models-of-ai-tools) for cross-tool comparison.

### 6.2 Building Your Own Agent Harness

Want to go beyond Claude Code's built-in orchestration? Build your own:

- **Agent loop** — a Python/TypeScript script that sends prompts to the Anthropic API, processes tool calls, and feeds results back
- **Context compression** — summarize old turns to stay within token limits
- **Git worktree management** — spin up isolated branches for each agent
- **Result aggregation** — collect outputs from parallel agents and merge

> **Repo to explore:** [Learn-Claude-Code](https://github.com/shareAI-lab/learn-claude-code) — build a Claude Code–style harness from scratch, covering agent loops, subagents, and context compression.

### 6.3 The Ninja Checklist

- [ ] You write CLAUDE.md files that make Claude follow your team's conventions without being told
- [ ] You have custom skills for every repetitive workflow (review, deploy, debug)
- [ ] You run MCP servers for your database, CI, and cloud provider
- [ ] You use hooks to auto-format, lint, and audit every change
- [ ] You orchestrate multi-agent teams for large refactors
- [ ] You've read Claude Code's system prompts and understand how they shape behavior
- [ ] You can build a custom agent harness from scratch
- [ ] Other developers ask *you* how to set up Claude Code

---

## Quick Reference: Repo → Level Mapping

| Repo | Best Level | Why |
|------|-----------|-----|
| [Claude-Code-Templates](https://github.com/davila7/claude-code-templates) | Beginner | Starter configs so you don't start from zero |
| [Claude-Code-Best-Practice](https://github.com/shanraisshan/claude-code-best-practice) | Intermediate | CLAUDE.md patterns, hooks, and commands |
| [Get-Shit-Done](https://github.com/gsd-build/get-shit-done) | Proficient | Structured workflow for plan → execute → verify |
| [Awesome-Claude-Code](https://github.com/hesreallyhim/awesome-claude-code) | Advanced | Ecosystem directory for skills, hooks, and plugins |
| [Awesome-Claude-Code-Subagents](https://github.com/VoltAgent/awesome-claude-code-subagents) | Expert | Specialized subagent definitions |
| [Claude Swarm](https://github.com/affaan-m/claude-swarm) | Expert | Multi-agent orchestration |
| [GStack](https://github.com/garrytan/gstack) | Expert | Role-based agent teams |
| [Claude-Code-System-Prompts](https://github.com/Piebald-AI/claude-code-system-prompts) | Ninja | Understanding internal prompts |
| [System-Prompts-and-Models-of-AI-Tools](https://github.com/x1xhlol/system-prompts-and-models-of-ai-tools) | Ninja | Cross-tool prompt comparison |
| [Learn-Claude-Code](https://github.com/shareAI-lab/learn-claude-code) | Ninja | Building your own harness from scratch |