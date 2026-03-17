---
description: "Generate go_router routes and guards for a feature flow"
argument-hint: "<feature flow description>"
---

# Generate Navigation Routes

## Instructions

You are generating go_router route definitions and navigation guards for a feature.

Parse `$ARGUMENTS` for the feature flow description.

Use the Agent tool with `subagent_type: "flutter-crossplatform-development:flutter-ui-expert"`:

```
Generate navigation routes for: $ARGUMENTS

## Context
Read the current project to find:
- Existing `GoRouter` configuration and route definitions
- Existing `ShellRoute`/`StatefulShellRoute` patterns
- Existing redirect guards (auth, onboarding, etc.)
- Screen widgets that need routes
- Named route conventions in use

## Deliverables
Write or update the following directly in the project:

1. **Route definitions** — `GoRoute` entries for each screen in the flow:
   - Named routes (no hardcoded path strings scattered in code)
   - Path parameters with `pathParameters` extraction
   - Query parameters where needed
   - `builder` or `pageBuilder` returning the screen widget wrapped in `BlocProvider`

2. **ShellRoute** (if applicable) — If the feature has shared UI (bottom nav, sidebar):
   - `ShellRoute` or `StatefulShellRoute` wrapping child routes
   - Shared scaffold with navigation bar

3. **Redirect guards** — If the feature requires authentication or other guards:
   - `redirect` callback checking auth state
   - Redirect to login/onboarding when guard fails
   - Preserve intended destination for post-login redirect

4. **Deep link support** — If the feature should be accessible via URL:
   - URI path design for deep linkability
   - Parameter extraction from URI

5. **Route constants** — Named route constants class or enum:
   - `FeatureRoutes.list`, `FeatureRoutes.detail`, etc.
   - Path builder methods for parameterized routes

6. **Navigation helpers** (optional) — Context extensions for common navigation:
   - `context.goToFeatureDetail(id)` wrapping `GoRouter.of(context).goNamed(...)`

Update the existing router configuration to include the new routes. Do not replace existing routes.
```

After the agent completes, show the user the route tree and any guards configured.
