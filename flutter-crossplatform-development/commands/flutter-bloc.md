---
description: "Generate a BLoC with sealed events, states, and explicit transformers"
argument-hint: "<bloc description> [--transformers sequential|droppable|restartable]"
---

# Generate BLoC

## Instructions

You are generating a complete BLoC implementation from a description.

Parse `$ARGUMENTS` for the BLoC description and any `--transformers` hint.

Use the Agent tool with `subagent_type: "flutter-crossplatform-development:flutter-feature-developer"`:

```
Implement a BLoC for: $ARGUMENTS

## Context
Read the current project structure to understand existing patterns, entity names, and repository interfaces. Match the conventions already in use.

## Deliverables
Write the following files directly into the project:

1. **Events file** — Sealed event hierarchy with action-oriented names and typed properties.
2. **States file** — Sealed state hierarchy with Equatable, `props`, shared base fields, and `isProcessing` getter where applicable.
3. **BLoC file** — Complete BLoC class with:
   - Constructor-injected repository interfaces
   - Every `on<>` handler with an explicit transformer
   - `switch` expression for logically grouped events sharing a transformer
   - `emit.forEach`/`onEach` with `restartable()` for stream subscriptions
   - Error handling: catch domain exceptions, emit error state, rethrow

4. **BLoC tests file** — Using `bloc_test`:
   - `blocTest` for every event → state transition
   - Transformer behavior verification (rapid events for droppable/restartable)
   - Error state tests with mocked repository throwing domain exceptions
   - Initial state test

Place files according to the project's existing directory structure. If no convention exists, use:
- `lib/features/<feature>/presentation/bloc/`
- `test/features/<feature>/presentation/bloc/`
```

After the agent completes, show the user a summary of files created and key design decisions (transformer choices, state hierarchy).
