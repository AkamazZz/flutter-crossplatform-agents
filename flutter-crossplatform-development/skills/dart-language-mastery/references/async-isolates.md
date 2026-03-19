---
description: >
  Async programming patterns with Future, Stream, Isolate.run, and concurrency best practices for Flutter.
---

# Async Programming and Isolates

---

## Future and async/await

`Future<T>` represents a single asynchronous value. `async`/`await` is the standard way to consume Futures.

```dart
Future<User> fetchUser(String id) async {
  final response = await http.get(Uri.parse('/users/$id'));
  if (response.statusCode != 200) throw HttpException(response.body);
  return User.fromJson(jsonDecode(response.body) as Map<String, dynamic>);
}
```

### Best Practices

- Always `await` Futures you care about. Unawaited Futures swallow exceptions silently. Use `unawaited()` from `package:meta` only when intentional fire-and-forget is documented.
- Prefer `async`/`await` over `.then()` chaining for readability.
- Return `Future<void>` for async methods that produce no value; do not return `Future<dynamic>` or `dynamic`.
- Do not use `async` on a function that has no `await` — it adds overhead and confuses readers.

### Error Handling

```dart
try {
  final user = await fetchUser(id);
  emit(AuthSuccess(user));
} on HttpException catch (e) {
  emit(AuthFailure(e.message));
} catch (e, st) {
  addError(e, st); // BLoC error sink
}
```

---

## Stream

`Stream<T>` represents a sequence of asynchronous values.

### Single-Subscription vs Broadcast

- Single-subscription — exactly one listener; file reads, HTTP response bodies, most data pipelines
- Broadcast — zero or more concurrent listeners; UI event streams, BLoC streams exposed to multiple widgets

```dart
// Single-subscription (default)
final stream = Stream.fromIterable([1, 2, 3]);

// Broadcast
final controller = StreamController<int>.broadcast();
```

A single-subscription stream that has already been listened to will throw if a second listener is added. Convert with `stream.asBroadcastStream()` when multiple listeners are needed.

---

## StreamController

`StreamController` lets you imperatively push events into a stream.

```dart
class LocationService {
  final _controller = StreamController<LatLng>.broadcast();

  Stream<LatLng> get positions => _controller.stream;

  void _onGpsUpdate(LatLng pos) => _controller.add(pos);

  Future<void> dispose() async {
    await _controller.close(); // always close to release resources
  }
}
```

### Lifecycle Rules

- Always `close()` the controller when the producing object is disposed.
- Check `_controller.hasListener` before adding if events before any listener would be wasteful.
- Use `onListen` and `onCancel` callbacks on single-subscription controllers to start/stop the underlying source.

---

## Isolate.run()

Dart is single-threaded per isolate. CPU-intensive work on the main isolate causes UI jank. `Isolate.run()` spawns a new isolate, runs the function, returns the result, then tears down the isolate.

```dart
Future<List<SearchResult>> runSearch(String query) async {
  final raw = await loadDataset(); // already in memory
  return Isolate.run(() => performSearch(raw, query));
}
```

The closure passed to `Isolate.run()` must only reference data that can be sent across isolate boundaries (primitives, plain Dart objects, `SendPort`). Closures that capture complex stateful objects (e.g., open files, platform channels) will fail.

### compute() — Deprecated

`compute()` was a Flutter convenience wrapper. It is deprecated in Flutter 3.7+. Replace all `compute(fn, message)` calls with `Isolate.run(() => fn(message))`.

### Long-Lived Isolates

For work that needs a persistent background thread (e.g., a background sync worker), use `Isolate.spawn()` with `SendPort`/`ReceivePort` message passing. `Isolate.run()` is for one-shot tasks only.

---

## Stream Transformers

Stream transformers encapsulate reusable stream transformation logic. Inline chaining is fine for trivial single-step transforms; anything more complex should be a named transformer (Global Rule 5).

### Built-In Transformers

```dart
stream
  .where((event) => event is UserAction)          // filter
  .map((event) => (event as UserAction).payload)  // transform
  .asyncMap((payload) => process(payload))        // async transform, preserves order
  .asyncExpand((payload) => fetchMultiple(payload)) // async transform, flattens
  .distinct()                                     // skip duplicates
  .debounceTime(...)                              // from rxdart
```

### asyncMap vs asyncExpand

- `asyncMap`: one input → one Future → one output. Order is preserved; backpressure is applied.
- `asyncExpand`: one input → one Stream → zero or more outputs. Use when one event fans out to multiple.

### Custom StreamTransformer

```dart
class DeduplicateTransformer<T> extends StreamTransformerBase<T, T> {
  @override
  Stream<T> bind(Stream<T> stream) async* {
    T? last;
    await for (final item in stream) {
      if (item != last) {
        yield item;
        last = item;
      }
    }
  }
}

// Usage
final deduplicated = rawStream.transform(DeduplicateTransformer());
```

---

## Zone Error Handling

Zones capture asynchronous errors that escape try/catch blocks.

```dart
runZonedGuarded(
  () => runApp(const MyApp()),
  (error, stack) {
    // Send to crash reporter
    FirebaseCrashlytics.instance.recordError(error, stack);
  },
);
```

`runZonedGuarded` wraps the entire app and catches uncaught async errors. This is the correct place to attach a global crash reporter. Do not rely on `FlutterError.onError` alone — it does not catch Zone errors from Dart async code.

### Zone-Local Values

Zones can carry values that are inherited by child zones. This is used internally by test frameworks and is rarely needed in application code.

```dart
final myKey = Object();
runZoned(
  () { /* zone value accessible here */ },
  zoneValues: {myKey: 'value'},
);
```

---

## Completer

`Completer<T>` bridges callback-based APIs into a Future.

```dart
Future<void> waitForPlatformReady() {
  final completer = Completer<void>();
  platformChannel.invokeMethod('isReady').then((_) {
    completer.complete();
  }, onError: completer.completeError);
  return completer.future;
}
```

**Rules:**
- Call `complete()` or `completeError()` exactly once. Calling either a second time throws.
- Use `completer.isCompleted` to guard against double-completion if the source can fire multiple times.
- Prefer async/await over Completer whenever the API supports it. Completer is for legacy callback APIs only.

---

## Common Mistakes

**Returning a Future from a void callback without awaiting.** The error is silently swallowed.

```dart
// Wrong
onPressed: () { saveData(); },

// Correct
onPressed: () async { await saveData(); },
```

**Using a single-subscription StreamController with multiple listeners.** The second `listen()` call throws. Use `.broadcast()`.

**Using `Isolate.run()` with a closure that captures a non-sendable object.** The isolate will fail at runtime with a `cannot send` error. Only plain data (JSON-like structures) crosses isolate boundaries.

**Forgetting to close a StreamController.** Leaked controllers hold references and may prevent garbage collection. Always close in `dispose()`.
