# CLAUDE.md

This is the `agency-slack` specialist plugin for Agency — an agent-first software development orchestrator.

## Purpose

This plugin provides Slack application specialist agents for two repositories:

- **webapp** — Slack's main web application monorepo (Hack backend + React/TypeScript frontend)
- **slack-desktop** — Slack's Electron-based desktop client (TypeScript, Redux, RxJS)

Unlike other Agency specialists that bring deep domain expertise, this plugin acts primarily as an **orchestration layer** — both repos (especially webapp) have extensive existing Claude Code tooling. The agents here understand how to navigate and leverage that tooling rather than duplicating it.

## Agents

- **slack-webapp-developer** — Implements tasks in the webapp monorepo. Leverages webapp's existing 50+ skills, 6 agents, MCP servers, and hooks.
- **slack-webapp-reviewer** — Reviews webapp implementations. Defers to webapp's specialized reviewers (a11y, security, etc.) for domain-specific concerns.
- **slack-desktop-developer** — Implements tasks in slack-desktop. Multi-process Electron expertise with Redux and RxJS epics.
- **slack-desktop-reviewer** — Reviews slack-desktop implementations. Focuses on multi-process correctness and Electron patterns.

## Skill Structure

Skills use a flat directory structure:

```
skills/
├── {skill-name}/
│   ├── SKILL.md           # Entry point with YAML frontmatter (required)
│   ├── references/        # Supporting reference docs (optional)
│   └── templates/         # Template directories (optional)
```

Skills are grouped by prefix:

| Prefix | Purpose |
|--------|---------|
| `webapp-` | Webapp-specific workflows: `webapp-navigation`, `webapp-devspaces`, `webapp-feature-flags` |
| `desktop-` | Desktop-specific patterns: `desktop-epics`, `desktop-preload`, `desktop-packaging` |
| `slack-` | Cross-repo Slack patterns (MCP servers, deploy, shared tooling) |

## Rules

| Rule | Scope | Purpose |
|------|-------|---------|
| `webapp-conventions.md` | webapp | Search policy, protected paths, UI requirements, git workflow |
| `desktop-conventions.md` | slack-desktop | Process boundaries, restart policy, tooling, state management |
| `slack-mcp-servers.md` | both | Available MCP servers and when to use each |

## Key Principle: Leverage, Don't Duplicate

The webapp repo already has extensive Claude Code tooling maintained by the webapp team. This plugin's agents should:

1. **Read the CLAUDE.md chain** in the target repo before starting any work
2. **Use existing skills** in the repo's `.claude/skills/` for repo-specific patterns
3. **Defer to existing agents** for specialized review (a11y, security, testing)
4. **Use existing MCP servers** (hakana, Slack Kit, etc.) for domain-specific lookups
5. **Only use Agency skills** for orchestration workflow and cross-repo patterns

## Agency Integration

Planning and product decisions are handled by Agency's orchestrator — not by this plugin. This plugin focuses on implementation expertise. Agents should:

1. Read their task file for instructions
2. Check `.agency/journals/slack-webapp.md` or `.agency/journals/slack-desktop.md` for project-specific decisions
3. Check feedback files for past corrections
4. Read the target repo's CLAUDE.md chain for current conventions
5. Use the target repo's existing skills and MCP servers for domain patterns
6. Follow the approach specified in the task file
