---
name: desktop-epics
description: RxJS epic patterns in slack-desktop — two coexisting systems (legacy createEpic + modern createAppEpic), custom operators, EpicTags debugging, and marble testing with rx-sandbox.
---

# Desktop Epics

All side effects in slack-desktop are implemented as epics — observable streams that react to Redux actions. The codebase has two coexisting systems with an active migration from legacy to modern.

## Two Epic Systems

### Modern: `createAppEpic` (preferred for new code)

Uses Redux Toolkit's listener middleware. Simpler, no RxJS required:

```typescript
import { createAppEpic } from '../common/epics/create-epic';
import { EpicTags } from '../common/constants/epics';

const myFeatureAppEpic = createAppEpic(EpicTags.MY_FEATURE, (epicApi) => {
  epicApi.addListener(someAction, ({ payload }, listenerApi) => {
    // Side effect logic
    listenerApi.dispatch(resultAction(payload.value));
  });
});

// Async variant — waits for environment to be ready
const myAsyncAppEpic = createAppEpic(
  EpicTags.MY_ASYNC_FEATURE,
  async ({ environmentReady, store }) => {
    await environmentReady;
    // Setup listeners that need the environment
    powerMonitor.on('resume', () => store.dispatch(powerResumed()));
  }
);
```

**`EpicAPI` interface:**
- `epicName: string` — the EpicTags value
- `store: Store<RootState>` — full Redux store
- `addListener(actionCreator, listener)` — typed listener registration
- `environmentReady: Promise<void>` — resolves when the app environment is initialized

### Legacy: `createEpic` (deprecated, do not use for new code)

RxJS observable streams with `redux-observable`:

```typescript
import { createEpic } from '../common/epics/create-epic';
import { ofType, completeAction, mapToAction } from '../common/custom-operators';
import { tag } from 'rxjs-spy/operators/tag';

// Side-effect-only epic (no output actions)
const myLegacyEpic = createEpic((actionObservable, stateObservable) => {
  return actionObservable.pipe(
    ofType(triggerAction),
    tag(EpicTags.MY_FEATURE),
    tap(({ payload }) => { /* side effect */ }),
    completeAction(EpicTags.MY_FEATURE)
  );
});

// Action-emitting epic
const myEmittingEpic = createEpic((actionObservable) => {
  return actionObservable.pipe(
    ofType(inputAction),
    tag(EpicTags.MY_EMITTING),
    mergeMap(({ payload }) => doAsyncWork(payload)),
    mapToAction((result) => outputAction(result), EpicTags.MY_EMITTING)
  );
});
```

## EpicTags Convention

Every epic MUST have a tag from the `EpicTags` enum in `src/common/constants/epics.ts`. Tag names end in `Epic`:

```typescript
export enum EpicTags {
  INITIALIZE_APPLICATION = 'initializeApplicationEpic',
  QUIT_APPLICATION = 'quitApplicationEpic',
  // ... 100+ entries
}
```

Tags serve three purposes:
1. **rxjs-spy debugging** — `tag(EpicTags.X)` in legacy epic pipes enables runtime tracing
2. **Structured logging** — used as prefixes: `logger.info(\`${EpicTags.X}: message\`)`
3. **Error tracking** — passed to `completeAction()`, `mapToAction()`, and `logEpicException()` for Sentry attribution

When adding a new epic, add a new `EpicTags` entry.

## Custom Operators

Defined in `src/common/custom-operators.ts`:

| Operator | Purpose |
|----------|---------|
| `ofType(...actionCreators)` | Filters action stream by type, returns typed output |
| `completeAction(tag)` | Terminates side-effect-only epics (logs errors, ignores elements) |
| `mapToAction(project, tag)` | Maps to output action with error logging |
| `logEpicException(tag)` | catchError that logs + reports to Sentry (prod) or re-throws (dev) |
| `guaranteedThrottle(time)` | switchMap + timer debounce pattern |

## Common RxJS Operators

Based on actual usage across the codebase:

- **`mergeMap`** — dominant flattening operator (concurrent async work)
- **`switchMap`** — cancels previous (used in window focus, notifications, preferences)
- **`tap`** — side effects (Electron API calls, logging)
- **`filter`** — action/state predicates
- **`take(1)`** — one-shot epics (quit, restart)
- **`race`** — timeout patterns (e.g., telemetry flush vs timer)
- **`delay` / `timer`** — time-based logic
- **`scan`** — accumulating state

Note: `exhaustMap` and `concatMap` are NOT used in the codebase.

## Epic Registration

Both systems register in `src/browser/epics/index.ts`:

```typescript
// Modern — array of epic functions
export const appEpics: Array<DesktopAppEpic> = [
  ...applicationAppEpics,
  ...browserWindowAppEpics,
  // ... ~20 more groups
];

// Legacy — combined via redux-observable
export const epics = combineEpics(
  ...applicationEpics,
  ...browserWindowEpics,
  // ... many more
  quitApplicationEpic  // deliberately last
);
```

Store wiring in `src/preload/redux/store.ts`:

```typescript
const epicApi = createEpicApi(store);
for (const appEpic of appEpics) {
  appEpic(epicApi);  // modern: just invoke
}
epicMiddleware.run(epics);  // legacy: run through middleware
```

## Testing Epics

### Modern app epics (preferred)

Use `DesktopTestStore` from `spec/jest/store-helper.ts`:

```typescript
import { setupTestStore } from '../../store-helper';
import { createEpicApi } from '../../common/epics/create-epic';

const store = setupTestStore({ preloadedState: defaultState });
const epicApi = createEpicApi(store);
epicApi.environmentReady = Promise.resolve();

it('dispatches expected action', async () => {
  myFeatureAppEpic(epicApi);
  store.dispatch(triggerAction(payload));
  await store.expectWillDispatch(expectedAction(value));
});

it('does not dispatch when condition fails', async () => {
  myFeatureAppEpic(epicApi);
  store.dispatch(wrongTrigger());
  store.expectNotDispatched(expectedAction);
});
```

**`DesktopTestStore` API:**
- `expectDispatched(matcher)` — sync check that action was dispatched
- `expectNotDispatched(matcher)` — sync check that action was NOT dispatched
- `expectWillDispatch(action)` — async, waits with timeout
- `expectWillNotDispatch(action)` — async negative
- `mergeState(partial)` — merge partial state
- `setState(full)` — replace full state
- `clearDispatched()` / `clear()` — reset

### Legacy epics (marble testing with rx-sandbox)

```typescript
import { rxSandbox } from 'rx-sandbox';
import { marbleAssert } from 'rx-sandbox/dist/assert/marbleAssert';
import { BehaviorStateObservable } from '../../__mocks__/behavior-state-observable';

let hot, expected, getMessages, scheduler;

beforeEach(() => {
  ({ scheduler, e: expected, hot, getMessages } = rxSandbox.create(true));
});

it('should emit restart when environment changes', () => {
  const action = hot('---x-x-y-', {
    x: handleDeepLink({ url: normalUrl }),
    y: handleDeepLink({ url: restrictedUrl }),
  });
  const value = getMessages(myEpic(action));
  marbleAssert(value).toEqual(expected('-------y-', { y: restartApp() }));
});

it('should not emit actions for side-effect-only epic', () => {
  const action = hot('---(xd)', { x: quitApp(), d: downloadsCleanedUp() });
  const value = getMessages(sideEffectEpic(action, stateObservable, { container, scheduler }));
  expect(value).toEqual(expected('---|'));
});
```

## Key Files

| File | Purpose |
|------|---------|
| `src/common/epics/create-epic.ts` | `createEpic`, `createAppEpic`, `createEpicApi`, type definitions |
| `src/common/constants/epics.ts` | `EpicTags` enum (100+ entries) |
| `src/common/custom-operators.ts` | `ofType`, `completeAction`, `mapToAction`, `logEpicException` |
| `src/browser/epics/index.ts` | Epic registration (both systems) |
| `src/browser/epics/epic-helpers.ts` | Shared utilities (`windowFocused`, `whenWindowExists`, etc.) |
| `src/browser/diagnostics/enable-epic-trace.ts` | rxjs-spy integration |
| `spec/jest/store-helper.ts` | `setupTestStore()` with `DesktopTestStore` |
| `spec/jest/mock-helper.ts` | Type aliases for rx-sandbox |
