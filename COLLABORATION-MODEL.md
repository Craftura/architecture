# Craftura Collaboration Model

> How Felippe (CTO) and Troy (CEO) work together with AI agents in 2026.

---

## The Problem

Troy is not a developer. He works on 5-10 projects simultaneously, thinks conversationally, and builds through exploration — not structured specs. His agents (Claude Code, Codex) spiral when left unsupervised, consuming compute and diverging from intent.

Felippe is the technical operator. He handles infrastructure, can run local models on RTX 5090, and manages the setup burden. His workflow with agents succeeds when there's a ticket to execute against — "first plan, then execute."

Both need to work on the same codebase without stepping on each other's changes.

## The Model: GitHub Issues as the Coordination Layer

### Core Principle

**GitHub Issues are the single source of truth for "what needs to be done."** Everything flows through issues.

### Workflow

```
Troy notices something needs to change
    → Creates a GitHub Issue (plain English, no technical specs)
        → Felippe (or Hermes) picks up the issue
            → Creates branch, implements, opens PR
                → Troy reviews at high level ("does this look right?")
                    → Merge to main
```

### Branching Strategy

```bash
# Each person/agent gets isolated worktrees — never clobber each other
git worktree add ../worktrees/felippe-12 -b felippe/issue-12
git worktree add ../worktrees/troy-explore -b troy/experiment
```

**Git worktrees** give each agent session its own isolated codebase. Same repository, different working directories. Parallel sessions never overwrite each other's changes.

### Issue Labels (for filtering)

| Label | Meaning | Who picks up |
|-------|---------|-------------|
| `frontend` | Web UI, pages, templates | Felippe/Hermes |
| `backend` | API, agents, pipelines | Felippe/Hermess |
| `design` | Visual polish, UX | Either |
| `marketing` | Copy, content, campaigns | Troy/Jade |
| `infra` | Deployment, servers, config | Felippe |
| `exploration` | Experimental, may not ship | Either |

### Rules

1. **Troy**: When you notice something needs to change, make an issue. Keep it in plain English — "the landing page needs better copy" is perfect. Don't worry about technical details.
2. **Felippe**: Pick up issues, implement, open PRs. Reference the issue number.
3. **Both**: Review each other's PRs before merging. Even a quick "looks good" review prevents drift.
4. **Agents**: Only work against an assigned issue. No feature creep without a ticket.
5. **Branch protection on `main`**: Requires PR review. No direct pushes.

### Why This Works

- **Troy doesn't need to know git** — he writes issues in plain English
- **Felippe handles the technical burden** — branches, PRs, CI/CD, agent config
- **Different tools don't matter** — Claude Desktop, Codex, Hermes all coordinate through GitHub
- **Async collaboration** — issues persist, PRs are reviewable anytime
- **Audit trail** — every change traces back to an issue

---

## Shared Agent Constitution

Every agent that touches Craftura repos reads a shared `AGENTS.md` at the repo root. This prevents identity hijacking and ensures consistent behavior regardless of which tool or model is being used.

### Required `AGENTS.md` content (per repo)

```markdown
# Craftura [Repo Name] — Agent Instructions

You are an AI assistant working on Craftura projects.
You do NOT assume any persona beyond this file.

## Team
- Felippe Burk (CTO): felippe@craftura.ai — technical lead, infrastructure
- Troy Stephens (CEO) — product direction, content, strategy

## Rules
1. Only work against assigned GitHub issues
2. Create branches: <username>/<short-description>
3. Open PRs for all changes — do not push to main directly
4. Ask questions via issue comments, not by making assumptions
5. Do NOT pretend to be any agent named Jade or any other persona

## Repository Structure
[Describe the repo layout here]

## Coding Standards
- [Language/framework conventions]
- [Testing requirements]
- [Commit message format]
```

This file is read by Claude Code, Codex, Cursor, Hermes — all major AI coding tools honor `AGENTS.md`/`CLAUDE.md`/`.cursorrules` at repo root.

---

## What Not To Do

| Anti-pattern | Why it fails | Alternative |
|--------------|-------------|-------------|
| Two agents editing same files in same directory | Silent corruption, context rot | Git worktrees — isolated directories |
| Direct pushes to main | No review, no audit trail | PRs with required review |
| Agent works without an issue | Feature creep, scope drift | "No ticket, no code" rule |
| Shared git account for humans + agents | Can't tell who did what | Separate accounts or clear attribution |
| Troy trying to branch/merge manually | Frustration, abandoned workflow | Issues only — Felippe handles git ops |

---

## Tool-Specific Notes

### Claude Code (Troy)
- Reads `CLAUDE.md` at repo root (symlink to `AGENTS.md`)
- Configure to require issue assignment before starting work
- Set up branch naming: `troy/<issue-number>-<description>`

### Codex (Troy)
- Same `AGENTS.md` pattern via `.github/copilot-instructions.md`
- Parallel sessions use separate worktrees

### Hermes (Felippe)
- Native GitHub integration for issue → branch → PR workflow
- Can auto-pick up assigned issues and create execution plans

---

## Daily Rhythm

| Time | Activity |
|------|----------|
| Morning | Check new issues, assign to yourself |
| Work session | Pick up an issue → branch → implement → PR |
| Before merging | Other person reviews (even if just "looks good") |
| End of day | Close completed issues, note any new ones that came up |

---

## Research Sources

- [obviousworks/agentic-coding-rulebook](https://github.com/obviousworks/agentic-coding-rulebook) — universal AGENTS.md standard
- [Git Worktrees for AI Coding](https://www.mindstudio.ai/blog/git-worktrees-parallel-ai-coding-agents) — parallel agent isolation
- [kururu-ai/autonomous-agent-playbook](https://github.com/kururu-ai/autonomous-agent-playbook) — human-agent collaboration patterns
- GitHub community discussions on multi-agent workflows (2025-2026)
