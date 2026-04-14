# Desktop Conventions

These rules apply when working in the `slack-desktop` repository. They supplement (not replace) the repo's own CLAUDE.md — always read it first.

## Process Boundaries

The desktop app has three process types. Code MUST respect these boundaries:

| Process | Can access | Cannot access |
|---------|-----------|---------------|
| Main (`src/browser/`) | Node.js APIs, Electron main APIs, file system | DOM, window objects |
| Preload (`src/preload/`) | Limited Node.js, `contextBridge` | Full Node.js, DOM directly |
| Renderer (`src/renderer/`) | DOM, `window.desktop` API | Node.js APIs, file system |

- Shared types, actions, and constants go in `src/common/`
- The preload API (`window.desktop`) is a contract — changes affect the hosted webapp

## Restart Required

Page reload is NOT sufficient after code changes. Always restart the Electron app to test changes.

## State Management

- Side effects in RxJS epics (`src/browser/epics/`), never in reducers
- Actions defined via Redux Toolkit (`createSlice`, `createAction`)
- New persisted state requires `redux-persist` configuration update
- Settings hierarchy: IT Policy > User Choices > IT Defaults > Slack Defaults

## Platform Awareness

- Guard platform-specific code with `process.platform` checks
- Native modules (`registry-js`, `cf-prefs`, etc.) are platform-specific — provide fallbacks
- Test on the platform you're targeting, or document platform-specific assumptions

## Tooling

- Package manager: Yarn 4 (Berry) only. Never use npm.
- Node.js: 22.13.1 (via `.nvmrc`)
- TypeScript: `tsgo` + `tsc` via `yarn run compile`
- Linting: ESLint via `yarn run lint`
- Formatting: Prettier via `yarn run format-all`
- Testing: Jest via `yarn test`

## Git Workflow

- Base branch: `master` (not `main`)
- Release branches: `release-#.#.x`
- CI: CircleCI (primary), GitHub Actions (auxiliary)
