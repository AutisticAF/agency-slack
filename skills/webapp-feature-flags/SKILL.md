---
name: webapp-feature-flags
description: Create and manage Houston feature flags in webapp — toggle generation, gating patterns, and cleanup.
---

# Feature Flags (Houston)

All new user-facing features in webapp should be gated behind Houston feature flags unless the task explicitly says otherwise.

## Creating a New Flag

1. **Create the flag in Houston**: `houston.tinyspeck.com`
2. **Generate the toggle class**:
   ```bash
   bin/gen-toggle <toggle_name>
   ```
   This generates a Hack class in `gen-hack/` (never modify generated code directly).

3. **Use the toggle in backend Hack code**:
   ```hack
   if (Toggle\YourToggleName::isEnabled($team_id)) {
     // new behavior
   }
   ```

4. **Use the toggle in frontend code**:
   Check the generated toggle interface for the frontend equivalent. Toggles are typically exposed via the boot data or experiment framework.

## Gating Patterns

**Backend (Hack):**
- Team-level: `Toggle\YourToggle::isEnabled($team_id)`
- User-level: `Toggle\YourToggle::isEnabledForUser($team_id, $user_id)`
- Org-level: Check Houston for org-level targeting

**Frontend (TypeScript):**
- Access via the experiments/feature flag framework in the boot data
- Check existing patterns in `js/modern/` for examples

## Flag Lifecycle

1. **Development**: Flag off by default, enabled for dev environments
2. **QA**: Enabled for QA environments and specific test teams
3. **Rollout**: Progressive percentage rollout via Houston
4. **Cleanup**: Remove flag and dead code path after full rollout

## Cleanup

Use webapp's existing `toggle-cleanup` Claude Code plugin for removing fully-rolled-out flags:
```
/toggle-cleanup <toggle_name>
```

This removes the toggle checks and dead code paths.

## Best Practices

- Name flags descriptively: `feature_name_short_description`
- Always have a clean off-path (existing behavior) and on-path (new behavior)
- Don't nest feature flags — if feature B depends on feature A, document the dependency
- Add the flag name to your task's Work Summary so reviewers can find it
