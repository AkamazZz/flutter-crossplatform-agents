---
description: "Generate bloc tests and golden tests for existing code"
argument-hint: "<file or feature path>"
---

# Generate Tests

## Instructions

You are generating bloc unit tests and golden file tests for existing code.

Parse `$ARGUMENTS` for the file path or feature name to test.

Use the Agent tool with `subagent_type: "flutter-crossplatform-development:flutter-testing-expert"`:

```
Generate tests for: $ARGUMENTS

## Context
Read the target code to find:
- All BLoC classes with their events, states, and transformers
- All screens/widgets that use BlocBuilder with sealed states
- Existing test patterns and directory structure in the project

## Deliverables
Write the following test files directly into the project:

1. **BLoC Unit Tests** — For each BLoC found, using `bloc_test`:
   - `blocTest` for every event → state transition path
   - Transformer behavior tests (rapid add() calls to verify sequential/droppable/restartable)
   - Error state tests with mocked repository throwing domain exceptions
   - Initial state test
   - `verify()` calls on mock repositories

2. **Golden File Tests** — For each screen/widget found:
   - One golden test per sealed state variant
   - Mock BLoC seeded with concrete state via `MockBloc` + `when(bloc.state).thenReturn(...)`
   - Wrapped in `MaterialApp` with theme for proper rendering
   - Deterministic font loading setup
   - File naming: `goldens/<feature>_<state_name>.png`

Place test files mirroring the source structure under `test/`.

After writing tests, run `flutter test` (excluding goldens) to verify bloc tests pass.
```

After the agent completes, show the user the test files created and coverage summary.
