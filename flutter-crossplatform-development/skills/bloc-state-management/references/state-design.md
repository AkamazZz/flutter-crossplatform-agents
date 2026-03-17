---
description: >
  BLoC State Design — sealed state hierarchy, Equatable props, convenience getters,
  state copy pattern, factory/named constructors, and state anti-patterns.
---

# State Design

> Cross-references: `bloc-core-rules.md` for architecture context and copy-data rule,
> `event-design.md` for the corresponding event hierarchy.

---

## 1. State Rules at a Glance

- Sealed class hierarchy — exhaustive switch in Widget and BLoC layers
- Extend `Equatable` — enables `==` and `hashCode` without manual overrides
- All fields `final` and marked `@immutable`
- Base class holds shared properties (items, cursor, hasMore)
- Subclasses add only the fields they uniquely need
- Always create a **new** state instance — never emit the same object reference
- Copy all unchanged fields forward on every transition

---

## 2. Full Sealed State Hierarchy Example

```dart
import 'package:equatable/equatable.dart';
import 'package:flutter/foundation.dart' show immutable;

part of 'feature_bloc.dart';

/// Base sealed class. Holds shared data so every substate always has it.
@immutable
sealed class FeatureState extends Equatable {
  const FeatureState({
    required this.items,
    this.cursor,
    this.hasMore = false,
  });

  final List<Item> items;
  final String? cursor;
  final bool hasMore;

  // -----------------------------------------------------------------------
  // Equatable
  // -----------------------------------------------------------------------

  /// Base props — all subclasses inherit these automatically.
  @override
  List<Object?> get props => [items, cursor, hasMore];

  // -----------------------------------------------------------------------
  // Convenience getters
  // -----------------------------------------------------------------------

  bool get isProcessing => this is FeatureStateProcessing;
  bool get hasError => this is FeatureStateError;
  bool get isIdle => this is FeatureStateIdle;
}

// ---------------------------------------------------------------------------

final class FeatureStateIdle extends FeatureState {
  const FeatureStateIdle({
    required super.items,
    super.cursor,
    super.hasMore,
  });

  // Inherits props from FeatureState — no override needed.
}

// ---------------------------------------------------------------------------

final class FeatureStateProcessing extends FeatureState {
  const FeatureStateProcessing({
    required super.items,
    super.cursor,
    super.hasMore,
  });
}

// ---------------------------------------------------------------------------

final class FeatureStateError extends FeatureState {
  const FeatureStateError({
    required this.message,
    required super.items,
    super.cursor,
    super.hasMore,
  });

  final String message;

  /// Override to add the extra field on top of base props.
  @override
  List<Object?> get props => [message, ...super.props];
}

// ---------------------------------------------------------------------------

/// State that carries newly fetched items before transitioning to Idle.
final class FeatureStateSuccess extends FeatureState {
  const FeatureStateSuccess({
    required super.items,
    super.cursor,
    super.hasMore,
  });
}
```

---

## 3. Equatable Props Pattern

### Rule: override `props` only when the subclass adds new fields

```dart
// ✅ Base class — define all shared fields here
sealed class FeatureState extends Equatable {
  const FeatureState({required this.items, this.cursor, this.hasMore = false});

  final List<Item> items;
  final String? cursor;
  final bool hasMore;

  @override
  List<Object?> get props => [items, cursor, hasMore];
}

// ✅ Subclass with no new fields — no override needed
final class FeatureStateIdle extends FeatureState {
  const FeatureStateIdle({required super.items, super.cursor, super.hasMore});
}

// ✅ Subclass WITH a new field — override using super.props spread
final class FeatureStateError extends FeatureState {
  const FeatureStateError({
    required this.message,
    required super.items,
    super.cursor,
    super.hasMore,
  });

  final String message;

  @override
  List<Object?> get props => [message, ...super.props];
}
```

**Why `...super.props`**: The spread ensures the base class fields are included without
manually repeating `items, cursor, hasMore` in every subclass override.

---

## 4. Convenience Getters

Declare type-check getters on the base class. The Widget layer and BLoC handlers can use
these without needing an `is` check inline.

```dart
sealed class FeatureState extends Equatable {
  // ...

  bool get isProcessing => this is FeatureStateProcessing;
  bool get hasError     => this is FeatureStateError;
  bool get isIdle       => this is FeatureStateIdle;

  // For states with extra fields, provide a typed cast getter:
  FeatureStateError? get asError =>
      this is FeatureStateError ? this as FeatureStateError : null;
}

// Widget usage (avoids verbose cast):
if (state.hasError) {
  final error = state.asError!;
  return ErrorWidget(message: error.message);
}
```

---

## 5. State Copy Pattern

**Every** state emission must carry forward all fields from the current state, even those
that did not change. Silent data loss is one of the most common BLoC bugs.

```dart
// ❌ WRONG — cursor and hasMore silently dropped
emit(FeatureStateProcessing(items: state.items));

// ✅ CORRECT — all fields explicitly forwarded
emit(FeatureStateProcessing(
  items: state.items,       // unchanged — still forwarded
  cursor: state.cursor,     // unchanged — still forwarded
  hasMore: state.hasMore,   // unchanged — still forwarded
));
```

### Named Constructor / Factory Approach

To make the copy pattern less verbose and harder to get wrong, provide named constructors
or factory methods on the base class:

```dart
sealed class FeatureState extends Equatable {
  const FeatureState({
    required this.items,
    this.cursor,
    this.hasMore = false,
  });

  final List<Item> items;
  final String? cursor;
  final bool hasMore;

  @override
  List<Object?> get props => [items, cursor, hasMore];

  // Named constructors as transition helpers
  FeatureStateProcessing processing() => FeatureStateProcessing(
        items: items,
        cursor: cursor,
        hasMore: hasMore,
      );

  FeatureStateIdle idle({List<Item>? items, String? cursor, bool? hasMore}) =>
      FeatureStateIdle(
        items: items ?? this.items,
        cursor: cursor ?? this.cursor,
        hasMore: hasMore ?? this.hasMore,
      );

  FeatureStateError error(String message) => FeatureStateError(
        message: message,
        items: items,
        cursor: cursor,
        hasMore: hasMore,
      );
}

// BLoC handler usage — concise and safe
Future<void> _onLoad(FeatureLoadEvent event, Emitter<FeatureState> emit) async {
  emit(state.processing());

  try {
    final data = await _repository.fetchData();
    emit(state.idle(items: data.items, cursor: data.cursor, hasMore: data.hasMore));
  } on Object catch (error, stackTrace) {
    developer.log('_onLoad failed', error: error, stackTrace: stackTrace);
    emit(state.error(error.toString()));
    rethrow;
  }
}
```

---

## 6. Anti-Patterns

### Mutable State Fields

```dart
// ❌ WRONG — non-final field can be mutated from outside the BLoC
final class FeatureStateIdle extends FeatureState {
  FeatureStateIdle({required this.items});
  List<Item> items;   // missing final
}

// ✅ CORRECT
final class FeatureStateIdle extends FeatureState {
  const FeatureStateIdle({required super.items, super.cursor, super.hasMore});
}
```

### Reference Mutation (Mutating a List In-Place)

```dart
// ❌ WRONG — mutates the list held by the current state; Equatable may not detect change
Future<void> _onDelete(FeatureDeleteEvent event, Emitter<FeatureState> emit) async {
  state.items.removeWhere((i) => i.id == event.itemId);  // mutation!
  emit(FeatureStateIdle(items: state.items, ...));
}

// ✅ CORRECT — create a new list
Future<void> _onDelete(FeatureDeleteEvent event, Emitter<FeatureState> emit) async {
  final updated = state.items.where((i) => i.id != event.itemId).toList();
  emit(FeatureStateIdle(items: updated, cursor: state.cursor, hasMore: state.hasMore));
}
```

### Identical States by Reference

```dart
// ❌ WRONG — emitting the exact same object Equatable already recorded
emit(state);   // Bloc ignores this; no rebuild triggered

// ✅ CORRECT — always construct a new instance
emit(FeatureStateIdle(
  items: state.items,
  cursor: state.cursor,
  hasMore: state.hasMore,
));
```

### Too Many States

```dart
// ❌ WRONG — a state per field change leads to combinatorial explosion
final class FeatureStateItemsLoaded extends FeatureState { ... }
final class FeatureStateItemsLoadedWithCursor extends FeatureState { ... }
final class FeatureStateItemsLoadedAndPaginating extends FeatureState { ... }
final class FeatureStateItemsLoadedCursorPaginating extends FeatureState { ... }

// ✅ CORRECT — shared base with flags/nullable fields; typically 3-4 states
sealed class FeatureState extends Equatable { ... }
final class FeatureStateIdle extends FeatureState { ... }
final class FeatureStateProcessing extends FeatureState { ... }
final class FeatureStateError extends FeatureState { ... }
```

Most BLoCs need exactly: **Idle**, **Processing**, **Error**, and occasionally **Success**
(a transient state that the BLoC immediately transitions away from).

### Storing Event Data in State

```dart
// ❌ WRONG — state remembers what triggered it; this is event information
final class FeatureStateProcessing extends FeatureState {
  const FeatureStateProcessing({
    required this.triggerEvent,   // ← never do this
    ...
  });
  final FeatureEvent triggerEvent;
}

// ✅ CORRECT — state holds data, not context about how it was reached
final class FeatureStateProcessing extends FeatureState {
  const FeatureStateProcessing({required super.items, super.cursor, super.hasMore});
}
```
