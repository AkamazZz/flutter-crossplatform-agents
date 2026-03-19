---
description: "Generate or update CompositionRoot wiring for a feature"
argument-hint: "<feature name>"
---

# Generate Dependency Injection Wiring

## Instructions

You are generating or updating the DI wiring for a feature.

Parse `$ARGUMENTS` for the feature name.

Use the Agent tool with `subagent_type: "flutter-crossplatform-development:flutter-feature-developer"`:

```
Generate the dependency injection wiring for feature: $ARGUMENTS

## Context
Read the current project to find:
- Existing `CompositionRoot`, `Dependencies`, `AppScope` classes
- Existing `*DependenciesBuilder` pattern and conventions
- All BLoCs, repositories, and data sources for this feature that need wiring
- How other features are already wired (match the pattern exactly)

## Deliverables
Write or update the following files directly in the project:

1. **FeatureDependenciesBuilder** — Static builder class that creates all dependencies for this feature:
   - Accepts core dependencies (Dio, Drift database, etc.) as parameters
   - Creates data sources, repositories, and BLoCs in correct order
   - Returns a `FeatureDependencies` container holding all created instances
   - Constructor injection only — no service locator

2. **FeatureDependencies container** — Immutable class holding all feature dependencies:
   - All fields `final`
   - Exposed to widgets via `context.featureXDependencies` extension

3. **CompositionRoot update** — Add the feature builder call to the existing CompositionRoot:
   - Wire core dependencies into the feature builder
   - Add the feature container to the `Dependencies` class

4. **AppScope update** — Add the feature's `RepositoryProvider`/`BlocProvider` entries to `MultiProvider` in AppScope (or the feature's local scope widget).

5. **Context extensions** — Add `context.featureXDependencies` extension for convenient access from widgets.

If the project has no existing DI pattern, create the full structure:
CompositionRoot → CoreDependenciesBuilder → FeatureDependenciesBuilder → Dependencies → AppScope with MultiProvider.
```

After the agent completes, show the user which files were created/updated and the dependency graph.
