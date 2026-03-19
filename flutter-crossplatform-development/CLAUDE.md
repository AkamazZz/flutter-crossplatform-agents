# Flutter Cross-Platform Development — Global Rules

These rules apply to ALL agents and all code in this project. Every agent must enforce the rules relevant to its domain.

## Rule 1 — Never use Cubit

Always use `Bloc` with explicit event processing. `Cubit` has no event processing order, no transformer support, and emits don't die after close. The only exceptions: a class that simply forwards a `Stream<T>` from a repository (no transformation), or an explicit user request for a learning/prototyping context. Treat any `Cubit` in production code as a bug to fix.

## Rule 2 — Constructor injection only

All dependencies flow through constructors and are stored as `final` private fields. Never use `GetIt`, `injectable`, `ComponentHolder`, or any global singleton registry. Flag `getIt<>()`, `locator<>()`, and `sl<>()` on sight. Platform channel wrappers, data sources, API clients, and database instances are all injected — never created internally.

## Rule 3 — No DTO or platform types in BLoC/Domain layer

The repository boundary maps DTOs to domain entities. DTOs never leave the data layer — every repository method returns domain entities, every mapper conversion happens inside the repository implementation. Platform-specific types (`PlatformException`, `DioException`, native data structures) never surface to BLoC or domain. Map to domain exceptions and entities at the repository boundary.

## Rule 4 — Sealed classes + Equatable

All states and events use sealed class hierarchies with concrete `final class` subtypes extending `Equatable`. States declare `props` — subclasses with extra fields extend via `[newField, ...super.props]`. Domain entities also use Equatable for value equality. All fields are `final` (immutable). Never use plain classes, enums as state machines, or classes without equality.

## Rule 5 — Explicit transformers on every `on<>` handler

Every `on<Event>(handler)` call must include an explicit `transformer:` parameter. A bare `on<Event>(handler)` without a transformer is a race condition waiting to happen. Common choices:
- `sequential()` — default, process events one at a time in arrival order
- `droppable()` — drop incoming events while one is processing
- `restartable()` — cancel current and start fresh; required for `emit.forEach`/`emit.onEach` stream subscriptions

## Rule 6 — No Hive, use Drift

For structured local data, always use Drift with type-safe tables and DAOs. For key-value data, use `SharedPreferences` (non-sensitive) or `flutter_secure_storage` (sensitive). Never recommend or use Hive.

## Rule 7 — `Theme.of(context)` for all visual properties

Zero tolerance for hardcoded colors, font sizes, font weights, or text styles. Every visual property comes from the theme via `Theme.of(context).colorScheme` and `Theme.of(context).textTheme`. If a design value is not in the standard Material 3 scheme, it belongs in a `ThemeExtension`. Flag `Colors.red`, `const TextStyle(fontSize: 14)`, and hex color literals immediately.

## Rule 8 — Localization classes for all strings

No user-facing string is hardcoded in a widget. All display text comes from generated localization classes (e.g., `AppLocalizations.of(context)!.loginButton`). Flag raw string literals in widget builds immediately.

## Rule 9 — `StatelessWidget` by default

If a widget doesn't manage ephemeral local state (no `AnimationController`, no `TextEditingController`, no `FocusNode`, no `PageController`), it is `StatelessWidget`. Converting to `StatefulWidget` requires explicit justification. BLoC state never justifies `StatefulWidget`.

## Rule 10 — `RepaintBoundary` for animated content, never `setState` for CustomPainter

Wrap every independently animated widget in `RepaintBoundary` to prevent its repaint from propagating to siblings. Pass `AnimationController` to `CustomPainter`'s `repaint:` parameter — `setState` in an animation listener rebuilds the entire widget subtree and is always wrong.

## Rule 11 — No `freezed` for BLoC states or events

BLoC states and events use hand-written sealed class hierarchies (Rule 4). `freezed` adds a code-generation dependency without benefit here. `freezed` is acceptable for complex DTOs or value objects outside the BLoC layer.

## Rule 12 — No `compute()`, use `Isolate.run()`

`compute()` is deprecated. Use `Isolate.run()` directly for CPU-bound offloading.

## Rule 13 — No `injectable`

`injectable` relies on `get_it` service locator, violating Rule 2. Use Pure DI with `CompositionRoot` instead.

---

## Directory Structure Convention

```
packages/
  domain/[feature]/lib/
    entities/[feature]_entity.dart
    repositories/[feature]_repository.dart

  data/[feature]/lib/
    repositories/[feature]_repository_impl.dart
    sources/
      remote/[feature]_api_client.dart
      local/[feature]_local_storage.dart
    models/[feature]_dto.dart
    mappers/[feature]_mapper.dart

lib/
  features/
    [feature]/
      bloc/[feature]_bloc.dart
      view/
        screens/[feature]_screen.dart
        widgets/[feature]_header.dart
```

## Flag On Sight

These patterns must be flagged and corrected immediately in any code:

- `getIt<>()`, `locator<>()`, `sl<>()`, `GetIt.instance`, `ServiceLocator.of()`
- `Cubit` in production code
- `on<Event>(handler)` without explicit `transformer:` parameter
- DTO class imported in BLoC or domain layer
- `Colors.red`, `Color(0xFF...)`, `TextStyle(fontSize: ...)` — hardcoded visual values
- `Hive`, `hive_flutter` in dependencies
- `freezed` on BLoC state or event classes
- `injectable` or `@injectable` annotations
- `compute()` calls — use `Isolate.run()`
- `dart:html` — use `dart:js_interop` + `package:web`
