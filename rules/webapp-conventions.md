# Webapp Conventions

These rules apply when working in the `webapp` repository. They supplement (not replace) webapp's own CLAUDE.md chain — always read `CLAUDE.md`, `src/CLAUDE.md`, and `js/CLAUDE.md` first.

## Search Policy

Never use the built-in Grep tool in webapp. The repo is too large and Grep results are unreliable.

- **Code search:** Use `rg` via Bash (ripgrep)
- **Hack symbols:** Use the `hakana` MCP server for symbol lookup, go-to-definition, and type information
- **UI components:** Use the Slack Kit MCP to look up available components

## Protected Paths

- `gen-hack/` — Generated Hack code. Never modify directly. Regenerate via the appropriate codegen tool.
- `translations/` — Managed externally. Claude Code permissions deny edits to this directory.

## Code Placement

- **New backend code** goes in `src/` using restructured Hack modules with namespaces
- **Never** add new code to `include/` (legacy) — only modify existing legacy code when the task explicitly requires it
- **Frontend code** in `js/modern/` for React components, `js/lib/` for shared libraries

## Formatting and Style

- **Indentation:** Tabs (both backend and frontend)
- **Backend formatting:** `hackfmt` — the auto-format hook handles this
- **Frontend formatting:** Prettier via `bin/fe fmt` — the auto-format hook handles this
- **Frontend linting:** ESLint with `@tinyspeck/slack/*` rules via `bin/fe lint`

## UI Components

All UI work MUST use Slack Kit components. Before building any UI:
1. Check Slack Kit MCP for available components
2. Use `--dt_*` design tokens for all visual properties
3. CSS Modules only — no inline styles, no global CSS
4. No hardcoded colors, spacing, or font sizes

## Git Workflow

- Base branch: `origin/master` (not local master, not `main`)
- Never force push on PRs with review activity
- Merge management: Checkpoint (`checkpoint.tinyspeck.com`)

## Dependencies

- Package manager: Yarn 4.x
- Exact versions only — no `^` or `~` in package.json
- Never add new third-party dependencies without task/RFC approval

## Feature Flags

- Use Houston (`houston.tinyspeck.com`) for feature gating
- Generate toggle classes: `bin/gen-toggle <toggle_name>`
- Always gate new user-facing features behind a flag unless the task says otherwise
