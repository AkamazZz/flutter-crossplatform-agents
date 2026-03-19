---
description: >
  BLoC Transformer Strategies — bloc_concurrency import, sequential/droppable/restartable
  patterns, emit.forEach/onEach for streams, logically scoped registration, and the
  "which transformer?" decision guide.
---

# Transformer Strategies

> Cross-references: `bloc-core-rules.md` Rule 5 (always explicit transformers),
> `event-design.md` Section 5 (event grouping maps to transformer selection).

---

## 1. Why Transformers Are Required

The default `on<Event>(handler)` registration in `flutter_bloc` processes events
**concurrently** — if two events of the same type arrive while one is still processing,
both handlers run in parallel. This is almost never what you want and is a source of
silent race conditions.

`bloc_concurrency` provides ready-made transformers that make the processing strategy
explicit and auditable.

**Always add an explicit `transformer:` argument to every `on<>` call.** No exceptions.

---

## 2. Import

```dart
import 'package:bloc_concurrency/bloc_concurrency.dart';
```

Add the package to `pubspec.yaml`:

```yaml
dependencies:
  bloc_concurrency: ^0.2.0
```

---

## 3. sequential() — Default, Most Common

### Behaviour

Events are queued and processed one at a time, in arrival order. The next event does not
start until the current handler completes (or emits its final state and returns).

### When to Use

- Any event group where order matters (most cases)
- Authentication flows (login → verify → redirect must not interleave)
- Data CRUD operations (create/update/delete must not overlap)
- Pagination (load-more events must not stack up and overlap)
- Form submissions

### Example

```dart
on<FeatureLoadEvents>(
  (event, emit) => switch (event) {
    FeatureLoadEvent() => _onLoad(event, emit),
    FeatureRefreshEvent() => _onRefresh(event, emit),
    FeatureLoadMoreEvent() => _onLoadMore(event, emit),
  },
  transformer: sequential(),
);
```

Full handler:

```dart
Future<void> _onLoad(
  FeatureLoadEvent event,
  Emitter<FeatureState> emit,
) async {
  emit(state.processing());

  try {
    final data = await _repository.fetchPage(cursor: null);
    emit(state.idle(items: data.items, cursor: data.cursor, hasMore: data.hasMore));
  } on Object catch (error, stackTrace) {
    developer.log('_onLoad', error: error, stackTrace: stackTrace);
    emit(state.error(error.toString()));
    rethrow;
  }
}
```

---

## 4. droppable() — Drop While Busy

### Behaviour

While a handler is running, any new events of the registered type (or group) are
silently **dropped**. The in-flight handler finishes normally.

### When to Use

- Single-shot actions: login button, purchase confirmation, jump in a game
- Aggregate fetches where only one request should ever be in-flight at a time
- Any action where re-triggering mid-flight produces incorrect behaviour

### Example

```dart
// ✅ Stats fetch — only one in-flight; extras dropped
on<FeatureFetchStatsEvent>(
  _onFetchStats,
  transformer: droppable(),
);

Future<void> _onFetchStats(
  FeatureFetchStatsEvent event,
  Emitter<FeatureState> emit,
) async {
  emit(state.processing());

  try {
    final stats = await _repository.fetchStats();
    emit(state.idle(items: stats.items, cursor: null, hasMore: false));
  } on Object catch (error, stackTrace) {
    developer.log('_onFetchStats', error: error, stackTrace: stackTrace);
    emit(state.error(error.toString()));
    rethrow;
  }
}
```

Login button example (prevents double-submit):

```dart
on<AuthSubmitEvent>(
  _onAuthSubmit,
  transformer: droppable(),
);
```

---

## 5. restartable() — Cancel Current, Start New

### Behaviour

When a new event arrives, the **current handler is cancelled** (its `Future` is abandoned;
any pending `await` stops) and a fresh handler starts for the new event.

### When to Use

- Search input / autocomplete — each keystroke should cancel the previous request
- Real-time suggestions, filtering, debounced queries
- Stream subscriptions via `emit.forEach` or `emit.onEach` — must be restartable so the
  subscription is torn down and restarted cleanly when the trigger event fires again

### Search Example

```dart
on<FeatureSearchEvent>(
  _onSearch,
  transformer: restartable(),
);

Future<void> _onSearch(
  FeatureSearchEvent event,
  Emitter<FeatureState> emit,
) async {
  // Debounce: wait 300ms; if cancelled before completion, this await is abandoned
  await Future<void>.delayed(const Duration(milliseconds: 300));

  emit(state.processing());

  try {
    final results = await _repository.search(query: event.query);
    emit(state.idle(items: results.items, cursor: null, hasMore: false));
  } on Object catch (error, stackTrace) {
    developer.log('_onSearch', error: error, stackTrace: stackTrace);
    emit(state.error(error.toString()));
    rethrow;
  }
}
```

---

## 6. emit.forEach and emit.onEach — Stream Subscriptions

Use these when a BLoC needs to **react to a repository stream** (e.g., authenticated user
changes, WebSocket messages, database change notifications). Always pair with `restartable()`.

### emit.forEach

```dart
on<_FeatureSubscribeEvent>(
  _onSubscribe,
  transformer: restartable(),
);

// Triggered once in the constructor: add(const _FeatureSubscribeEvent());
Future<void> _onSubscribe(
  _FeatureSubscribeEvent event,
  Emitter<FeatureState> emit,
) =>
    emit.forEach<User?>(
      _authRepository.userStream,
      onData: (user) => FeatureStateIdle(
        items: state.items,
        cursor: state.cursor,
        hasMore: state.hasMore,
      ),
      onError: (error, stackTrace) {
        developer.log('_onSubscribe stream error', error: error, stackTrace: stackTrace);
        return state.error(error.toString());
      },
    );
```

### emit.onEach

`emit.onEach` is identical to `emit.forEach` but lets you call `emit` manually, which is
useful when the stream item alone is not a complete state:

```dart
Future<void> _onSubscribe(
  _FeatureSubscribeEvent event,
  Emitter<FeatureState> emit,
) =>
    emit.onEach<List<Item>>(
      _repository.itemStream,
      onData: (items) => emit(FeatureStateIdle(
        items: items,
        cursor: state.cursor,
        hasMore: state.hasMore,
      )),
      onError: (error, stackTrace) {
        developer.log('stream error', error: error, stackTrace: stackTrace);
        emit(state.error(error.toString()));
      },
    );
```

### Why restartable() is mandatory for streams

If the subscription event fires a second time (e.g., BLoC is reused after reconnect),
`restartable()` cancels the old subscription before starting the new one. Without it,
two concurrent subscriptions would both emit states, causing unpredictable interleaving.

---

## 7. Logically Scoped Registration — Full Example

```dart
class FeatureBloc extends Bloc<FeatureEvent, FeatureState>
    with _FeatureStateEmitter {
  FeatureBloc({
    required IFeatureRepository repository,
    required IAuthRepository authRepository,
  })  : _repository = repository,
        _authRepository = authRepository,
        super(const FeatureStateIdle(items: [], cursor: null, hasMore: false)) {

    // --- 1. Internal stream subscription: restartable ------------------
    on<_FeatureSubscribeEvent>(
      _onSubscribe,
      transformer: restartable(),
    );

    // --- 2. Stats fetch: droppable -------------------------------------
    on<FeatureFetchStatsEvent>(
      _onFetchStats,
      transformer: droppable(),
    );

    // --- 3. Read / paginate events: sequential -------------------------
    on<FeatureLoadEvents>(
      (event, emit) => switch (event) {
        FeatureLoadEvent() => _onLoad(event, emit),
        FeatureRefreshEvent() => _onRefresh(event, emit),
        FeatureLoadMoreEvent() => _onLoadMore(event, emit),
      },
      transformer: sequential(),
    );

    // --- 4. Mutation events: sequential --------------------------------
    on<FeatureActionEvents>(
      (event, emit) => switch (event) {
        FeatureDeleteEvent() => _onDelete(event, emit),
        FeatureSubmitEvent() => _onSubmit(event, emit),
      },
      transformer: sequential(),
    );

    // --- 5. Search: restartable ----------------------------------------
    on<FeatureSearchEvent>(
      _onSearch,
      transformer: restartable(),
    );

    // Kick off the stream subscription immediately
    add(const _FeatureSubscribeEvent());
  }

  final IFeatureRepository _repository;
  final IAuthRepository _authRepository;

  // ... handler implementations ...
}
```

---

## 8. Decision Guide — Which Transformer?

```
Does the event start a long-running operation (async/stream)?
│
├── No (synchronous, fire-and-forget)
│   └── sequential()
│
└── Yes
    │
    ├── Should new events DROP while one is in flight?
    │   (action cannot be repeated mid-flight: login, jump, purchase)
    │   └── droppable()
    │
    ├── Should new events CANCEL the current one and restart?
    │   (search input, autocomplete, stream subscriptions)
    │   └── restartable()
    │
    └── Should events QUEUE and run one at a time?
        (CRUD operations, pagination, auth flow)
        └── sequential()
```

Quick cheat-sheet:

- Most event groups (default) → `sequential()`
- Load / refresh / paginate → `sequential()`
- Auth flow (login, logout, verify) → `sequential()`
- CRUD (create, update, delete) → `sequential()`
- Login button, purchase, game jump → `droppable()`
- Aggregate stats fetch → `droppable()`
- Search input, autocomplete → `restartable()`
- Repository stream subscription → `restartable()`
- Real-time filter / debounce → `restartable()`

---

## 9. Anti-Patterns

### Bare `on<Event>(handler)` Without Transformer

```dart
// ❌ WRONG — concurrent by default; silent race condition
MyBloc() : super(initialState) {
  on<LoadEvent>(_onLoad);
  on<RefreshEvent>(_onRefresh);
}

// ✅ CORRECT
MyBloc() : super(initialState) {
  on<FeatureLoadEvents>(
    (event, emit) => switch (event) {
      LoadEvent() => _onLoad(event, emit),
      RefreshEvent() => _onRefresh(event, emit),
    },
    transformer: sequential(),
  );
}
```

### Using restartable() for Mutations

```dart
// ❌ WRONG — a delete cancelled mid-flight may leave the backend in an inconsistent state
on<FeatureDeleteEvent>(_onDelete, transformer: restartable());

// ✅ CORRECT
on<FeatureActionEvents>(
  (event, emit) => switch (event) {
    FeatureDeleteEvent() => _onDelete(event, emit),
    ...
  },
  transformer: sequential(),
);
```

### Using droppable() for Pagination

```dart
// ❌ WRONG — droppable drops load-more taps while fetching, losing pagination requests
on<FeatureLoadMoreEvent>(_onLoadMore, transformer: droppable());

// ✅ CORRECT — queue them; each page loads after the previous completes
on<FeatureLoadEvents>(
  (event, emit) => switch (event) {
    FeatureLoadMoreEvent() => _onLoadMore(event, emit),
    ...
  },
  transformer: sequential(),
);
```

### Missing restartable() on Stream Subscriptions

```dart
// ❌ WRONG — without restartable(), re-triggering the subscribe event creates
// two concurrent subscriptions emitting states in parallel
on<_SubscribeEvent>(_onSubscribe);   // no transformer — race condition

// ✅ CORRECT
on<_SubscribeEvent>(_onSubscribe, transformer: restartable());
```
