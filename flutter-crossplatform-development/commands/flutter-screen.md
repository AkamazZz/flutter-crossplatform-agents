---
description: "Generate a screen widget wired to an existing BLoC"
argument-hint: "<screen description> [--bloc ExistingBlocName]"
---

# Generate Screen

## Instructions

You are generating a complete screen widget connected to a BLoC.

Parse `$ARGUMENTS` for the screen description and optional `--bloc` flag.

Use the Agent tool with `subagent_type: "flutter-crossplatform-development:flutter-ui-expert"`:

```
Build a screen for: $ARGUMENTS

## Context
Read the current project to find:
- The BLoC class this screen connects to (use --bloc hint if provided, otherwise infer from the feature)
- The sealed state hierarchy to build the exhaustive switch
- Existing theming, navigation, and widget patterns in the project

## Deliverables
Write the following files directly into the project:

1. **Screen widget** — StatelessWidget (default) with:
   - `BlocProvider` at the top providing the BLoC
   - `BlocBuilder` with exhaustive `switch` on sealed state subtypes
   - `BlocListener` for side effects (navigation, snackbar, dialog)
   - `buildWhen` on complex builders
   - `Theme.of(context)` for all colors and text styles
   - Localization classes for all user-facing strings

2. **Extracted private widgets** — Private `StatelessWidget` classes for complex subtrees (no `_buildX()` methods).

3. **Context extensions** — If the screen has repeated `context.read<Bloc>().add(Event())` patterns, create a `BuildContext` extension.

4. **go_router route** — Show the route definition to add to the router configuration.

5. **Golden tests** — One golden test per sealed state variant:
   - Mock BLoC seeded with each state
   - Wrapped in MaterialApp with theme
   - Deterministic font loading

Place files according to the project's existing directory structure.
```

After the agent completes, show the user the files created and which states are rendered.
