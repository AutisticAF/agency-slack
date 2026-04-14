---
name: desktop-preload
description: The window.desktop preload bridge in slack-desktop — architecture, namespaces, IPC patterns, and step-by-step guide for adding new methods.
---

# Desktop Preload Bridge

The preload bridge (`window.desktop`) is the contract between the Electron shell and the hosted webapp at `app.slack.com`. Changes here affect both repos.

## Architecture

Two communication directions:

- **Renderer -> Main**: Webapp calls `window.desktop.namespace.method()`, preload translates to `ipcRenderer.invoke()` / `ipcRenderer.send()`, main process handles with `ipcMain.handle()` / `ipcMain.on()`
- **Main -> Renderer**: Main process calls `webContents.send(IpcMainMessage.*)`, preload receives via `ipcRenderer.on()`, forwards to `window.desktopDelegate.callback()`

## Bridge Initialization

Entry point: `src/preload/bootstrap-util/initialize-main-preload.ts`

1. `getDesktopPreloadObject(store)` builds the full namespace object
2. Assigned to `window.desktop` (and legacy `window.winssb`)
3. `contextBridge.exposeInMainWorld('desktop', ...)` exposes to isolated webapp context
4. Webapp registers its callback delegate via `window.desktop.setGlobalDelegate(delegate)`

Component windows (notifications, settings, etc.) get a smaller API via `src/preload/component-preload-entry-point.ts`.

## Namespaces on `window.desktop`

Defined in `src/common/interfaces/preload-api/desktop-interface.ts`:

| Namespace | Purpose |
|-----------|---------|
| `accessibility` | Screen reader support |
| `app` | App lifecycle, version, preferences |
| `calls` | Huddle/call integration |
| `clipboard` | Read/write clipboard |
| `contextMenu` | Right-click context menus |
| `contextMenus` | Menu registration |
| `diagnostics` | Logging, debug info |
| `dock` | macOS dock badge/bounce |
| `downloads` | File download management |
| `localFileSearch` | OS file search |
| `menus` | Application menu bar |
| `notice` | Native notifications |
| `recentFiles` | OS recent files list |
| `redux` | Direct Redux store access |
| `search` | Search index integration |
| `screen` | Screen/display info |
| `callsScreenshare` | Screen sharing for calls |
| `spellCheckerV3` | Spell checking |
| `stats` | Telemetry/metrics |
| `teams` | Multi-workspace management |
| `window` | Window management |
| `workspaces` | Workspace switching |
| `system` | OS-level integration |

Plus `windowChrome` (Windows-only / debug) — not part of public `DesktopInterface`.

27 interface files in `src/common/interfaces/preload-api/`.

## IPC Channel Names

Defined in `src/common/constants/ipc-channel-names.ts`:

- **`IpcRendererMessage`** — ~100+ channels for renderer-to-main (e.g., `WRITE_STRING_TO_CLIPBOARD`, `GET_SCREEN_PREVIEW_THUMBNAILS`)
- **`IpcMainMessage`** — ~12 channels for main-to-renderer (e.g., `FORWARD_DEEP_LINK`, `START_SEARCH`, `RELOAD_DOCUMENT`)

## The Delegate Pattern (Main -> Renderer)

The webapp registers a `DesktopDelegate` callback object. When the main process needs to notify the renderer, the preload forwards to the delegate:

```typescript
// Preload listener (register-preload-handlers.ts)
ipcRenderer.on(IpcMainMessage.START_SEARCH, () => {
  window.desktopDelegate?.startSearch();
});
```

`DesktopDelegate` interface (`src/common/interfaces/preload-api/delegate.ts`) defines ~25 callbacks: `handleDeepLinkWithArgs`, `startSearch`, `closeCall`, `openDialog`, `focusNextWorkspace`, `reload`, `showUpdateBanner`, etc.

## Example: Clipboard Namespace (Simplest Pattern)

**Interface** (`src/common/interfaces/preload-api/clipboard.ts`):
```typescript
export interface ClipboardInterface {
  writeString: (text: string) => Promise<void>;
  writeImage: (base64EncodedPngOrJpg: string) => Promise<void>;
}
```

**Preload implementation** (`src/preload/desktop-interface/clipboard.ts`):
```typescript
export const createClipboardInterface = (): ClipboardInterface => ({
  writeString: (text) => ipcInvoke(IpcRendererMessage.WRITE_STRING_TO_CLIPBOARD, text),
  writeImage: (base64) => ipcInvoke(IpcRendererMessage.WRITE_IMAGE_TO_CLIPBOARD, base64),
});
```

**Main handler** (registered in `src/browser/register-main-process-handlers.ts`):
```typescript
ipcHandle(IpcRendererMessage.WRITE_STRING_TO_CLIPBOARD, (_, text) => {
  clipboard.writeText(text);
});
```

## Adding a New Method to the Preload API

Follow these steps in order:

### 1. Define the IPC channel name
`src/common/constants/ipc-channel-names.ts` — add to `IpcRendererMessage` (renderer->main) or `IpcMainMessage` (main->renderer).

### 2. Define the TypeScript interface
Add to an existing interface in `src/common/interfaces/preload-api/` or create a new file. If creating a new namespace, add it to `DesktopInterface` in `desktop-interface.ts`.

### 3. Add argument type mapping
`src/common/interfaces/ipc-renderer-interface-args.ts` — map your new `IpcRendererMessage` to the parameter types.

### 4. Add Joi validation schema
`src/browser/ipc-validator/ipc-renderer-interface.ts` — add your channel with Joi validators for each argument. This protects against malicious IPC calls.

### 5. Implement the preload function
`src/preload/desktop-interface/<namespace>.ts` — call `ipcInvoke` (request/response) or `ipcSend` (fire-and-forget).

### 6. Register the main-process handler
`src/browser/register-main-process-handlers.ts` — use `ipcHandle(channel, handler)` for invoke-style or `ipcListen(channel, handler)` for send-style. Both wrappers validate origin + arguments automatically.

### 7. For main->renderer messages (optional)
- Add `ipcRenderer.on(IpcMainMessage.*)` listener in `src/preload/desktop-interface/register-preload-handlers.ts`
- Add the callback to `DesktopDelegate` in `src/common/interfaces/preload-api/delegate.ts`

### 8. Update child window support (if applicable)
Update `@slack/desktop/is-api-available` in the webapp (per the comment at line 149 of `initialize-main-preload.ts`).

## Security

- `validateEventOrigin` in `src/browser/ipc-from-renderer.ts` checks that the IPC sender is from a valid Slack URL origin
- Joi schemas validate all IPC arguments before the handler runs
- Never expose raw Node.js APIs through the bridge

## Key Files

| File | Purpose |
|------|---------|
| `src/preload/bootstrap-util/initialize-main-preload.ts` | Bridge initialization (main windows) |
| `src/preload/component-preload-entry-point.ts` | Bridge initialization (component windows) |
| `src/preload/desktop-interface/get-desktop-preload-object.ts` | Desktop object factory |
| `src/common/interfaces/preload-api/desktop-interface.ts` | Master `DesktopInterface` definition |
| `src/common/interfaces/preload-api/*.ts` | All 27 namespace interface files |
| `src/common/interfaces/preload-api/delegate.ts` | `DesktopDelegate` (main->renderer callbacks) |
| `src/preload/types/global-augment.ts` | Global `Window` type augmentation |
| `src/common/constants/ipc-channel-names.ts` | IPC channel name enums |
| `src/common/interfaces/ipc-renderer-interface-args.ts` | IPC argument type mappings |
| `src/preload/ipc-to-main.ts` | Typed `ipcInvoke`/`ipcSend` helpers |
| `src/preload/desktop-interface/register-preload-handlers.ts` | Main->renderer listener registration |
| `src/browser/register-main-process-handlers.ts` | Main-process handler registration |
| `src/browser/ipc-from-renderer.ts` | `ipcHandle`/`ipcListen` wrappers + origin validation |
| `src/browser/ipc-validator/ipc-renderer-interface.ts` | Joi argument validators |
