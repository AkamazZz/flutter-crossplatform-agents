---
description: "Refactor code to fix architecture violations"
argument-hint: "[file or feature path]"
---

# Refactor Architecture Violations

## Instructions

You are refactoring existing code to fix violations of the project's global rules and architectural standards.

If `$ARGUMENTS` is provided, refactor that specific file or feature. If empty, refactor all recently changed files.

## Step 1: Run review first

Before refactoring, run the `/flutter-review` checklist against the target code. Identify all violations with file paths, line numbers, and the specific rule broken.

Present the violations to the user and ask:

```
Found N violations:

1. [Rule X] file.dart:line — description
2. [Rule Y] file.dart:line — description
...

1. Fix all — refactor everything
2. Select — let me choose which to fix
3. Cancel — don't change anything
```

Wait for user response before proceeding.

## Step 2: Fix violations by category

Group fixes by type and dispatch to the appropriate agent:

**BLoC violations (Rules 1, 4, 5)** — Use `subagent_type: "flutter-crossplatform-development:flutter-state-expert"`:
- Convert Cubit to Bloc
- Add missing transformers to `on<>` handlers
- Convert plain classes to sealed hierarchies with Equatable

**Data layer violations (Rules 2, 3, 6)** — Use `subagent_type: "flutter-crossplatform-development:flutter-data-engineer"`:
- Replace service locator calls with constructor injection
- Move DTO usage behind repository boundary
- Replace Hive with Drift

**UI violations (Rules 7, 8, 9, 10)** — Use `subagent_type: "flutter-crossplatform-development:flutter-ui-expert"`:
- Replace hardcoded colors/styles with Theme.of(context)
- Replace hardcoded strings with localization
- Convert unjustified StatefulWidget to StatelessWidget

**Architecture violations** — Use `subagent_type: "flutter-crossplatform-development:flutter-architect"`:
- Fix domain layer importing data/Flutter packages
- Untangle cross-feature imports
- Move wiring from widget tree to CompositionRoot

Run agents in parallel where fixes are independent (different files).

## Step 3: Verify

After all fixes, run `/flutter-review` again on the changed files to confirm zero violations remain.
