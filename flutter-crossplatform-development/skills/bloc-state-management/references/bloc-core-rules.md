---
description: >
  BLoC Core Rules — architecture layers, file structure, BLoC implementation patterns,
  and critical anti-patterns. Read this first for any BLoC-related work.
---

# BLoC Core Rules

> **Global Rules Callout**
> - **Rule 1**: Never use `Cubit` — use `Bloc` exclusively (see Section 7)
> - **Rule 2**: Constructor injection only — never instantiate or resolve repositories inside BLoC (see Section 7)
> - **Rule 4**: Sealed classes + Equatable for all states and events — see `state-design.md` and `event-design.md`
> - **Rule 5**: Always declare explicit transformers on every `on<>` registration — see `transformer-strategies.md`

---

## 1. Core Principles

### What BLoC Is

- **Business Logic Component**: Separates the Widget layer and the Data layer using a Pub/Sub pattern
- **State Machine**: Provides a predictable sequence of event-to-state transformations
- **Race Condition Prevention**: Manages processing order to avoid inconsistent states
- **Decentralized**: Multiple BLoCs for different purposes — not a single centralized store
- **Declarative**: Expresses asynchronous sequences of state changes

### What BLoC Is NOT

- Not a function from event to single state — one event can produce zero, one, or infinitely many states
- Not a ViewModel or Presenter
- Not a presentation layer — it is Business *Logic* Component
- Not data storage scoped to a single screen ("one BLoC per screen" is an incorrect mental model)
- Not aware of implementation details — must never expose database models or DTOs to the Widget layer

---

## 2. Architecture Layers

```
Widget Layer ←→ BLoC Layer ←→ Domain Layer ←→ Data Layer
(UI/Config)     (Business Logic)  (Entities/Interfaces)  (Repositories/Sources)
```

### Widget Layer
- Interacts with BLoC **only** through: `add()`, `stream`, `state`
- Contains: UI configuration, navigation, localisation, lifecycle hooks
- Business logic errors must never reach widgets unprocessed

### BLoC Layer
- Transforms events into states using current state and repository interfaces
- Manages business workflow and processing order via transformers
- Bridges Widget and Domain/Data layers
- No `flutter/…` imports (except `package:flutter/foundation.dart`)

### Domain Layer
- Abstract repository interfaces returning entities or primitives
- Pure Dart — no Flutter, no JSON annotations, no third-party framework coupling

### Data Layer
- `Repository impl → Data Provider → Client / Database`
- Repositories orchestrate multiple data providers
- Data providers map DTOs to domain entities at the boundary

---

## 3. File Structure

### Single-File Approach (Preferred for Small/Medium Features)

```dart
// feature_bloc.dart
part 'feature_state_event.dart';

// States, Events, and BLoC all in one file for cohesion
```

### Multi-File Approach (Complex Features)

```dart
// feature_bloc.dart
part 'feature_event.dart';
part 'feature_state.dart';
```

Use multi-file only when the single file exceeds ~200 lines or when state/event types are
numerous enough to make navigation difficult.

---

## 6. BLoC Implementation

### Constructor Injection

Repositories **must** be passed via the constructor and stored as `final` fields.

```dart
class FeatureBloc extends Bloc<FeatureEvent, FeatureState> {
  FeatureBloc({
    required IFeatureRepository repository,
  })  : _repository = repository,
        super(const FeatureStateIdle(items: [], cursor: null, hasMore: false)) {
    on<FeatureLoadEvents>(
      (event, emit) => switch (event) {
        FeatureLoadEvent() => _onLoad(event, emit),
        FeatureRefreshEvent() => _onRefresh(event, emit),
      },
      transformer: sequential(),
    );
  }

  final IFeatureRepository _repository;
}
```

### Immutability

All repository fields are `final`. BLoC classes carry no mutable instance variables beyond
what the `state` getter already tracks.

### State as Storage

Use the `state` getter to read data between handler calls. Do **not** introduce private
instance variables (`_items`, `_cursor`, etc.) to shadow what state already holds.

```dart
// ❌ WRONG — private variable duplicates state
class FeatureBloc extends Bloc<FeatureEvent, FeatureState> {
  List<Item> _items = [];   // duplicates state.items

  Future<void> _onLoad(FeatureLoadEvent event, Emitter<FeatureState> emit) async {
    _items = await _repository.fetchItems();
    emit(FeatureStateIdle(items: _items, ...));
  }
}

// ✅ CORRECT — read from state
Future<void> _onLoad(FeatureLoadEvent event, Emitter<FeatureState> emit) async {
  emit(FeatureStateProcessing(
    items: state.items,
    cursor: state.cursor,
    hasMore: state.hasMore,
  ));
  final data = await _repository.fetchItems();
  emit(FeatureStateIdle(
    items: data.items,
    cursor: data.cursor,
    hasMore: data.hasMore,
  ));
}
```

### No Public Methods

The only public surface of a BLoC is: `add()`, `stream`, `state`, and `close()`.
Never add extra public methods, getters, or setters.

### No Flutter Imports

Starting from the BLoC layer downward, do not import `flutter/…` packages (except
`package:flutter/foundation.dart` for `@immutable` and `debugPrint`).

### Error Handling Pattern

```dart
Future<void> _onLoad(
  FeatureLoadEvent event,
  Emitter<FeatureState> emit,
) async {
  emit(FeatureStateProcessing(
    items: state.items,
    cursor: state.cursor,
    hasMore: state.hasMore,
  ));

  try {
    final data = await _repository.fetchData();
    emit(FeatureStateIdle(
      items: data.items,
      cursor: data.cursor,
      hasMore: data.hasMore,
    ));
  } on Object catch (error, stackTrace) {
    developer.log(
      'FeatureBloc: failed to load',
      error: error,
      stackTrace: stackTrace,
    );
    emit(FeatureStateError(
      message: error.toString(),
      items: state.items,
      cursor: state.cursor,
      hasMore: state.hasMore,
    ));
    rethrow; // let BlocObserver see it
  }
}
```

---

## 7. Critical Anti-Patterns

### Never Use Cubit

Cubit has no event queue — there is no processing order, which means race conditions are
possible by design. Cubit uses observer pattern, not pub/sub. Emits do not die after `close()`.

**Exceptions** (rare, deliberate): a pure stream relay (e.g., WebSocket presence indicator)
or an explicit learning-phase proof-of-concept. Document the exception with a comment.

### Never Create Repository Inside BLoC

```dart
// ❌ WRONG
class MyBloc extends Bloc<MyEvent, MyState> {
  final _repo = MyRepository();            // direct instantiation
  // or:
  final _repo = getIt<MyRepository>();     // service locator inside BLoC
}

// ✅ CORRECT
class MyBloc extends Bloc<MyEvent, MyState> {
  MyBloc({required IMyRepository repository})
      : _repository = repository,
        super(const MyStateIdle());

  final IMyRepository _repository;
}
```

### Never Make BLoCs Depend on Each Other

```dart
// ❌ WRONG
class CartBloc extends Bloc<CartEvent, CartState> {
  CartBloc({required AuthBloc authBloc}) : _authBloc = authBloc, super(...);
  final AuthBloc _authBloc;
}

// ✅ CORRECT — both depend on a shared repository; use emit.forEach with restartable()
class CartBloc extends Bloc<CartEvent, CartState> {
  CartBloc({required IAuthRepository authRepository})
      : _authRepository = authRepository,
        super(const CartStateIdle()) {
    on<_SubscribeToAuthEvent>(
      _onSubscribeToAuth,
      transformer: restartable(),
    );
  }

  final IAuthRepository _authRepository;

  Future<void> _onSubscribeToAuth(
    _SubscribeToAuthEvent event,
    Emitter<CartState> emit,
  ) =>
      emit.forEach(
        _authRepository.userStream,
        onData: (user) => CartStateIdle(user: user),
      );
}
```

### Never Register `on<>` Without an Explicit Transformer

```dart
// ❌ WRONG — no transformer, silent race condition risk
MyBloc() : super(initialState) {
  on<EventA>(_onEventA);
  on<EventB>(_onEventB);
}

// ✅ CORRECT — always explicit
MyBloc() : super(initialState) {
  on<EventA>(_onEventA, transformer: sequential());
  on<EventB>(_onEventB, transformer: droppable());
}
```

---

## 8. Best Practices

### Sealed Classes for Events and States

Always declare events and states as `sealed class`. This enables exhaustive `switch`
expressions in both the BLoC and the Widget layer with no default fallthrough.

```dart
sealed class FeatureEvent { const FeatureEvent(); }
sealed class FeatureState extends Equatable { ... }
```

See `event-design.md` for full event patterns and `state-design.md` for state hierarchy.

### Copy All Data Between State Transitions

Every state constructor call must carry forward all fields from the current state, even
those that did not change. This prevents silent data loss.

```dart
// ✅ CORRECT — all fields forwarded
emit(FeatureStateProcessing(
  items: state.items,      // unchanged — still forwarded
  cursor: state.cursor,    // unchanged — still forwarded
  hasMore: state.hasMore,  // unchanged — still forwarded
));
```

### Cross-References

- Event naming, grouping, and State Emitter mixin → `event-design.md`
- State hierarchy, Equatable props, convenience getters → `state-design.md`
- Transformer selection (sequential/droppable/restartable) → `transformer-strategies.md`
