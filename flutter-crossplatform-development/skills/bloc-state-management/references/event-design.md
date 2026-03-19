---
description: >
  BLoC Event Design — sealed event class patterns, action-oriented naming, State Emitter
  mixin, event grouping with switch expressions, and event anti-patterns.
---

# Event Design

> Cross-references: `bloc-core-rules.md` for architecture context, `transformer-strategies.md`
> for how event grouping maps to transformer selection.

---

## 1. Event Rules at a Glance

- Sealed class hierarchy — enables exhaustive switch expressions
- Mark the base class `@immutable`; subclasses use `final` fields
- Action-oriented names: **Load, Refresh, Delete, Submit** — NOT Loaded, Refreshed, Deleted
- Group related events under a sealed parent when they share a transformer
- Use State Emitter mixin to avoid duplicating `emit` calls across handlers
- Never put business logic inside event classes — events are data carriers only

---

## 2. Full Sealed Event Hierarchy Example

```dart
import 'package:flutter/foundation.dart' show immutable;

part of 'feature_bloc.dart';

/// Base sealed class for all FeatureBloc events.
@immutable
sealed class FeatureEvent {
  const FeatureEvent();
}

// ---------------------------------------------------------------------------
// Group: read/fetch events — will share sequential() transformer
// ---------------------------------------------------------------------------

/// Parent for all load-related events.
sealed class FeatureLoadEvents extends FeatureEvent {
  const FeatureLoadEvents();
}

/// Triggered when the screen opens for the first time.
final class FeatureLoadEvent extends FeatureLoadEvents {
  const FeatureLoadEvent();
}

/// Pull-to-refresh: reload from page 1.
final class FeatureRefreshEvent extends FeatureLoadEvents {
  const FeatureRefreshEvent();
}

/// Pagination: load the next cursor page.
final class FeatureLoadMoreEvent extends FeatureLoadEvents {
  const FeatureLoadMoreEvent();
}

// ---------------------------------------------------------------------------
// Group: mutation events — will share sequential() transformer
// ---------------------------------------------------------------------------

sealed class FeatureActionEvents extends FeatureEvent {
  const FeatureActionEvents();
}

final class FeatureDeleteEvent extends FeatureActionEvents {
  const FeatureDeleteEvent({required this.itemId});
  final String itemId;
}

final class FeatureSubmitEvent extends FeatureActionEvents {
  const FeatureSubmitEvent({required this.payload});
  final SubmitPayload payload;
}

// ---------------------------------------------------------------------------
// Standalone event — will use droppable() transformer
// ---------------------------------------------------------------------------

/// Fetch aggregate statistics; drop duplicates while processing.
final class FeatureFetchStatsEvent extends FeatureEvent {
  const FeatureFetchStatsEvent();
}

// ---------------------------------------------------------------------------
// Internal / private events (prefixed with _) — not part of public API
// ---------------------------------------------------------------------------

/// Subscribe to repository stream updates.
final class _FeatureSubscribeEvent extends FeatureEvent {
  const _FeatureSubscribeEvent();
}
```

---

## 3. Action-Oriented Naming Conventions

Correct (action verb) → Incorrect (past/noun form):
- `FeatureLoadEvent` not `FeatureLoadedEvent`
- `FeatureRefreshEvent` not `FeatureRefreshedEvent`
- `FeatureDeleteEvent` not `FeatureItemDeletedEvent`
- `FeatureSubmitEvent` not `FeatureFormSubmittedEvent`
- `FeatureFetchStatsEvent` not `FeatureStatsFetchEvent`

Events name the **action the user or system is requesting**, not the result that occurred.
Results are expressed through state transitions, not through event names.

---

## 4. State Emitter Mixin Pattern

When multiple event handlers repeat the same emit sequence (e.g., always emit Processing
then Idle or Error), extract that sequence into a mixin to remove duplication.

```dart
/// Mixin that provides convenience emit helpers for FeatureBloc handlers.
mixin _FeatureStateEmitter on Bloc<FeatureEvent, FeatureState> {
  /// Emit a processing state carrying all current data forward.
  void emitProcessing(Emitter<FeatureState> emit) => emit(
        FeatureStateProcessing(
          items: state.items,
          cursor: state.cursor,
          hasMore: state.hasMore,
        ),
      );

  /// Emit an error state carrying all current data forward.
  void emitError(Emitter<FeatureState> emit, String message) => emit(
        FeatureStateError(
          message: message,
          items: state.items,
          cursor: state.cursor,
          hasMore: state.hasMore,
        ),
      );

  /// Return to idle carrying all current data forward.
  void emitIdle(Emitter<FeatureState> emit) => emit(
        FeatureStateIdle(
          items: state.items,
          cursor: state.cursor,
          hasMore: state.hasMore,
        ),
      );
}

// Usage in BLoC:
class FeatureBloc extends Bloc<FeatureEvent, FeatureState>
    with _FeatureStateEmitter {
  // ...

  Future<void> _onLoad(
    FeatureLoadEvent event,
    Emitter<FeatureState> emit,
  ) async {
    emitProcessing(emit);   // ← mixin call instead of inline emit

    try {
      final data = await _repository.fetchData();
      emit(FeatureStateIdle(
        items: data.items,
        cursor: data.cursor,
        hasMore: data.hasMore,
      ));
    } on Object catch (error, stackTrace) {
      developer.log('FeatureBloc._onLoad failed', error: error, stackTrace: stackTrace);
      emitError(emit, error.toString());   // ← mixin call
      rethrow;
    }
  }

  Future<void> _onRefresh(
    FeatureRefreshEvent event,
    Emitter<FeatureState> emit,
  ) async {
    emitProcessing(emit);   // ← same helper, no duplication

    try {
      final data = await _repository.fetchData(cursor: null);
      emit(FeatureStateIdle(
        items: data.items,
        cursor: data.cursor,
        hasMore: data.hasMore,
      ));
    } on Object catch (error, stackTrace) {
      developer.log('FeatureBloc._onRefresh failed', error: error, stackTrace: stackTrace);
      emitError(emit, error.toString());
      rethrow;
    }
  }
}
```

---

## 5. Event Grouping with Switch Expression

Group events that share a transformer under a sealed parent class, then use a single `on<>`
registration with a `switch` expression to dispatch to individual handlers.

```dart
FeatureBloc({required IFeatureRepository repository})
    : _repository = repository,
      super(const FeatureStateIdle(items: [], cursor: null, hasMore: false)) {

  // Standalone event with its own transformer
  on<FeatureFetchStatsEvent>(
    _onFetchStats,
    transformer: droppable(),
  );

  // Group: all load-related events share sequential()
  on<FeatureLoadEvents>(
    (event, emit) => switch (event) {
      FeatureLoadEvent() => _onLoad(event, emit),
      FeatureRefreshEvent() => _onRefresh(event, emit),
      FeatureLoadMoreEvent() => _onLoadMore(event, emit),
    },
    transformer: sequential(),
  );

  // Group: all mutation events share sequential()
  on<FeatureActionEvents>(
    (event, emit) => switch (event) {
      FeatureDeleteEvent() => _onDelete(event, emit),
      FeatureSubmitEvent() => _onSubmit(event, emit),
    },
    transformer: sequential(),
  );

  // Internal stream subscription with restartable()
  on<_FeatureSubscribeEvent>(
    _onSubscribe,
    transformer: restartable(),
  );

  add(const _FeatureSubscribeEvent());
}
```

The `switch` expression is exhaustive — if a new event subclass is added to the sealed
hierarchy without a corresponding case, the compiler reports an error immediately.

---

## 6. Anti-Patterns

### Events Carrying Too Much Data

```dart
// ❌ WRONG — event carries precomputed or derived data
final class FeatureLoadEvent extends FeatureEvent {
  const FeatureLoadEvent({
    required this.items,          // items should come from repository, not from widget
    required this.totalCount,
    required this.formattedDate,
  });
  final List<Item> items;
  final int totalCount;
  final String formattedDate;
}

// ✅ CORRECT — event carries only the minimal trigger signal or user input
final class FeatureLoadEvent extends FeatureEvent {
  const FeatureLoadEvent();
}

final class FeatureDeleteEvent extends FeatureEvent {
  const FeatureDeleteEvent({required this.itemId});
  final String itemId;           // only the identifier; BLoC fetches/derives the rest
}
```

### Events Named After State Changes (Past Tense)

```dart
// ❌ WRONG — past tense implies the action already happened
final class ItemsLoadedEvent extends FeatureEvent { ... }
final class FormSubmittedEvent extends FeatureEvent { ... }

// ✅ CORRECT — imperative, present tense
final class FeatureLoadEvent extends FeatureEvent { ... }
final class FeatureSubmitEvent extends FeatureEvent { ... }
```

### Non-Sealed Event Classes

```dart
// ❌ WRONG — abstract base, not sealed; switch is non-exhaustive
abstract class FeatureEvent { ... }

// ❌ WRONG — no base class; events are unrelated, no grouping possible
class FeatureLoadEvent { ... }
class FeatureRefreshEvent { ... }

// ✅ CORRECT — sealed, enabling exhaustive switch
sealed class FeatureEvent { const FeatureEvent(); }
```

### Mutable Event Fields

```dart
// ❌ WRONG — event fields must be final (immutable)
final class FeatureSubmitEvent extends FeatureEvent {
  FeatureSubmitEvent({required this.payload});
  SubmitPayload payload;   // non-final — can be mutated after dispatch
}

// ✅ CORRECT
final class FeatureSubmitEvent extends FeatureEvent {
  const FeatureSubmitEvent({required this.payload});
  final SubmitPayload payload;
}
```
