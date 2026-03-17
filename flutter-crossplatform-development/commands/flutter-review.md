---
description: "Review code against global rules and flag violations"
argument-hint: "[file or feature path]"
---

# Review Flutter Code

## Instructions

You are reviewing Flutter code against the project's global rules and architectural standards.

If `$ARGUMENTS` is provided, review that specific file or feature directory. If empty, review all recently changed files (`git diff --name-only HEAD~1`).

Read the target code, then check for every violation:

## Checklist

### Rule 1 — No Cubit
- [ ] Any `Cubit` class usage → must be `Bloc`

### Rule 2 — Constructor injection only
- [ ] Any `getIt<>()`, `locator<>()`, `sl<>()`, `GetIt.instance`
- [ ] Any dependency created inside a class instead of injected

### Rule 3 — No DTO/platform types in BLoC/Domain
- [ ] Any DTO import in BLoC or presentation files
- [ ] Any `DioException`, `PlatformException` caught in BLoC layer
- [ ] Any `json_serializable` annotation in domain entities

### Rule 4 — Sealed classes + Equatable
- [ ] Events or states using plain classes instead of sealed hierarchies
- [ ] Missing `Equatable` or `props` override
- [ ] Mutable fields (non-final) in states

### Rule 5 — Explicit transformers
- [ ] Any `on<Event>(handler)` without `transformer:` parameter
- [ ] Stream subscriptions (`emit.forEach`/`onEach`) without `restartable()`

### Rule 6 — No Hive
- [ ] Any `hive` or `hive_flutter` import

### Rule 7 — Theme.of(context)
- [ ] Hardcoded `Colors.*`, hex color literals, `TextStyle(fontSize: ...)`

### Rule 8 — Localization
- [ ] Hardcoded user-facing strings in widget builds

### Rule 9 — StatelessWidget default
- [ ] `StatefulWidget` without ephemeral state justification (no controllers, no animation)

### Rule 10 — RepaintBoundary
- [ ] `setState` in animation listeners
- [ ] `CustomPainter` without `repaint:` parameter
- [ ] `Paint()` created inside `paint()` method

### Architecture
- [ ] Domain layer importing data or Flutter packages
- [ ] BLoC importing `package:flutter/*` (except `foundation.dart`)
- [ ] Cross-feature BLoC or widget imports
- [ ] BLoC-to-BLoC injection (instead of shared repository streams)

## Output Format

For each violation found, report:
```
VIOLATION: Rule X — [rule name]
File: path/to/file.dart:line_number
Found: [the offending code]
Fix: [the corrective code or approach]
```

If no violations found, confirm the code passes all checks.
