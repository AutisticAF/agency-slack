# Slack MCP Servers

Available MCP servers for Slack development. These provide domain-specific capabilities beyond standard Claude Code tools.

## Webapp MCP Servers

Configured in webapp's `.mcp.json`:

**`hakana`** — Hack language server
- Symbol lookup, go-to-definition, type information
- Use instead of Grep for finding Hack functions, classes, and types
- Local server: `hakana-mcp-server`

**`slack`** — Slack platform MCP
- HTTP MCP at `https://mcp.slack.com/mcp`
- OAuth-authenticated

**Slack Kit MCP** — Design system components
- Installed via `slack-claude plugin install slack-kit-mcp`
- Look up available UI components before building any UI
- Required for all frontend UI work in webapp

## Discovery

- Internal MCP directory: `socs.tinyspeck.com/mcp-servers`
- DevXP AI catalog: `ai.tinyspeck.com/mcp-servers`
- Add servers: `claude mcp add --transport http <name> <url>`
- Scoping: `--scope local` (just you), `--scope project` (shared via `.mcp.json`), `--scope user` (all projects)

## DevSpaces MCP

When working in DevSpaces (remote dev), additional MCP servers are available:

**`hd_exec`** — Execute commands inside DevSpace containers
- Required for running HHVM, Hack type checker, and backend tests in DevSpaces

**`git-gh`** — Git/GitHub CLI for DevSpaces
- Custom MCP in webapp's `.claude/mcp/git-gh.js`
- Required in sandbox mode (DevSpaces)
