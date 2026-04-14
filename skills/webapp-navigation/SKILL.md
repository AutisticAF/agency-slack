---
name: webapp-navigation
description: Navigate webapp's CLAUDE.md chain and existing Claude Code tooling to find the right agent, skill, or MCP server for the task at hand.
---

# Webapp Tooling Navigation

The webapp repo has one of the most extensive Claude Code configurations at Slack. Before starting work, use this skill to find the right existing tools.

## CLAUDE.md Chain

Always read these in order before writing code:

1. **`CLAUDE.md`** (root) — Project structure, search policies, git workflow, cross-cutting patterns
2. **`src/CLAUDE.md`** (backend) — Hack conventions, module structure, testing, DB patterns. Read before ANY backend work.
3. **`js/CLAUDE.md`** (frontend) — React patterns, Slack Kit, CSS Modules, design tokens. Read before ANY frontend work.

## Finding the Right Existing Agent

Webapp has 6 specialized agents in `.claude/agents/`. Check if your task maps to one:

| Agent | Use When |
|-------|----------|
| `a11y-reviewer` | Task involves accessibility or ARIA |
| `backend-caching-expert` | Task involves caching layer or memoization |
| `hacklang-test-writer` | Need to write Hack tests |
| `playwright-test-evaluator` | Need to write or evaluate E2E tests |
| `rtl-test-writer` | Need to write React Testing Library tests |
| `security-reviewer` | Task touches auth, permissions, or sensitive data |

## Finding the Right Existing Skill

Webapp has 50+ skills in `.claude/skills/`. Key categories:

- **Accessibility**: a11y audit, label patterns
- **Testing**: Hack BDD tests, RTL tests, Playwright, TDD
- **Migration**: XHP-to-React, JS-to-TS
- **Feature flags**: Houston experiment setup, toggle cleanup
- **Design**: Figma-to-code, Slack Kit patterns
- **Security**: Risk assessment, ACL patterns
- **AI**: Quality evals, experiment review

To discover available skills:
```bash
ls .claude/skills/
```

## Finding the Right MCP Server

| Need | MCP Server | How to Use |
|------|-----------|------------|
| Hack symbol lookup | `hakana` | Type/function/class search in Hack code |
| UI components | Slack Kit MCP | Look up design system components |
| Slack platform APIs | `slack` | OAuth-authenticated Slack API access |
| Code search (DevSpaces) | `hd_exec` | Run `rg` inside DevSpace container |

## Existing Hooks (Auto-Run)

These fire automatically — no action needed:
- **auto-format.py** — Formats code after every edit (hackfmt for Hack, Prettier for JS/TS)
- **fix-curly-quotes.py** — Replaces curly quotes with straight quotes

## Plugin Marketplace

Webapp uses the `slack-claude-code-plugins` marketplace. Installed plugins:
- `fix-ci` — CI failure diagnosis
- `code-search` — Enhanced code search
- `toggle-cleanup` — Feature flag cleanup
- `slack-kit-mcp` — Slack Kit design system
