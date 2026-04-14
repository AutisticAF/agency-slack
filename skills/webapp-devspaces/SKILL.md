---
name: webapp-devspaces
description: Work with Slack DevSpaces for webapp backend development — setup, container execution, and common workflows.
---

# DevSpaces Workflow

DevSpaces are remote development environments required for webapp backend (Hack/HHVM) development. HHVM is not supported locally on macOS.

## When to Use DevSpaces

- **Required**: All backend Hack development, backend tests, HHVM execution
- **Optional**: Frontend work (local dev is faster on M-series Macs)
- **Fullstack**: Local frontend (`bin/fe build client` on port 1989) + DevSpace backend

## Setup

```bash
# Attach a branch to a DevSpace
slack remote-dev --branch <BRANCH_NAME>

# SSH into your DevSpace
slack ssh-dev --env=devNNNN

# Access via browser
# https://devNNNN.slack.com/client
```

## Running Commands in DevSpaces

Use `hd_exec` MCP server (available in sandbox mode) to run commands inside the webapp container:

```
# Inside DevSpace container
hd_exec yarn install
hd_exec bin/hh          # Hack type checker
hd_exec hackfmt <file>  # Hack formatter
hd_exec ./tests/run_unit_in_curl.sh  # Backend tests
```

## DevSpace Lifecycle

- DevSpaces have a **1-week lifespan** (extendable once for 1 additional week)
- Multiple containers run per DevSpace: `webapp`, `webapp_test_server`, `job_worker`
- Code is mounted at `/mnt/devNNN` on the remote machine

## Claude Code in DevSpaces

When using Claude Code inside a DevSpace:
- Sandbox mode is active — network restricted to `slack-github.com`
- Use `git-gh` MCP server for git/GitHub operations
- See `.claude/CLAUDE-DEVSPACES.md` for DevSpaces-specific guidance

## Fullstack Development

Combine local frontend with remote backend:

1. Start local frontend build: `bin/fe build client` (port 1989)
2. Access DevSpace with local assets: `https://devNNNN.slack.com/client?js_path=1`
3. Frontend changes reflect immediately; backend changes require DevSpace container restart

## Common Issues

- **DevSpace not responding**: Check `#escal-developer-workspaces`
- **HHVM errors**: Run inside the container via `hd_exec`, not on the host
- **Git in DevSpaces**: Use `git-gh` MCP, not direct git commands (sandbox restrictions)
