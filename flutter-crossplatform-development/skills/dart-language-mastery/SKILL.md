---
name: dart-language-mastery
description: >
  Advanced Dart language features including patterns, records, sealed classes, async programming,
  and code generation. Use when working with Dart 3 features, async patterns, Isolates, or
  build_runner code generation. Trigger on: Dart 3, patterns, records, sealed classes, switch
  expressions, async, Future, Stream, Isolate, code generation, build_runner, freezed.
---

# Dart Language Mastery

## Overview

Dart 3.x introduced a foundational shift in how Flutter applications are structured. Patterns, records, and sealed classes provide exhaustive, type-safe modeling of application state. Async primitives (Future, Stream, Isolate) drive responsive UI and background computation. Code generation via build_runner eliminates boilerplate for serialization and routing.

These features are not isolated conveniences — they form the architectural backbone of the patterns used throughout this project: sealed class hierarchies replace ad-hoc nullable fields; switch expressions replace fragile if/else chains; Isolate.run() replaces compute(); and json_serializable replaces manual fromJson/toJson.

## Reference Files

| File | Topics | When to Consult |
|------|--------|-----------------|
| [patterns-records-sealed.md](references/patterns-records-sealed.md) | Pattern matching, destructuring, records, sealed classes, guard clauses, logical/relational patterns | Modeling states, events, results; exhaustive switching; multiple return values |
| [async-isolates.md](references/async-isolates.md) | Future, Stream, StreamController, Isolate.run(), Zone, Completer, stream transformers | Any async code, background work, stream pipelines, callback bridging |
| [code-generation.md](references/code-generation.md) | build_runner, json_serializable, freezed, go_router gen, injectable (anti-pattern), custom builders | Setting up or running code gen, serialization, routing generation |

## Quick Reference

| Feature | Syntax | When to Use |
|---------|--------|-------------|
| Switch expression | `result = switch (x) { pattern => value, ... }` | Replace if/else chains; exhaustive over sealed types |
| Record literal | `(name, age)` or `(name: 'Ada', age: 36)` | Return multiple values without defining a class |
| Record destructuring | `final (name, age) = record;` | Unpack record fields inline |
| Object pattern | `case User(:final name, :final age)` | Destructure class fields in switch arms |
| Guard clause | `case Foo(:final x) when x > 0` | Add a boolean condition to a pattern arm |
| Sealed class | `sealed class State {}` + subclasses | Model closed sets of states/events with exhaustiveness |
| Isolate.run() | `await Isolate.run(() => heavyWork())` | Offload CPU-intensive work off the main thread |
| Stream broadcast | `controller.stream` (broadcast) | Multiple listeners on the same stream |
| Completer | `final c = Completer<T>(); ... c.complete(value);` | Bridge a callback API into a Future |
| asyncMap | `stream.asyncMap((e) async => ...)` | Async transformation in a stream pipeline |

## Anti-Patterns

**Do not use nullable fields as a substitute for sealed classes.** A state object with `String? errorMessage` and `bool isLoading` is an uncontrolled product type. Use a sealed class hierarchy instead — every consumer is forced to handle every case at compile time.

**Do not nest switch statements.** If an inner switch is needed, the sealed hierarchy is likely under-specified. Flatten by adding more specific subclasses.

**Do not use `compute()`.** It is deprecated. Use `Isolate.run()` directly.

**Do not use `freezed` for BLoC states or events.** freezed generates equality and copyWith, but BLoC states and events are modeled as sealed class hierarchies (Global Rule 4). Introducing freezed here adds a code-generation dependency to the core domain layer without benefit. freezed is acceptable for complex DTOs or value objects that live outside the BLoC layer.

**Do not use `injectable`.** It relies on the get_it service locator, which violates Global Rule 2 (constructor injection only). See [code-generation.md](references/code-generation.md) for a full explanation.

## Global Rules (Relevant)

### Rule 4 — Sealed Classes for States and Events

All BLoC states and events must be modeled as sealed class hierarchies. Each subclass is a concrete, named case. Switch expressions over the sealed type must be exhaustive — the compiler enforces that every case is handled. No default arms are permitted unless the arm explicitly covers a wildcard that is intentional and documented.

```dart
sealed class AuthState {}
final class AuthInitial extends AuthState {}
final class AuthLoading extends AuthState {}
final class AuthSuccess extends AuthState {
  const AuthSuccess(this.user);
  final User user;
}
final class AuthFailure extends AuthState {
  const AuthFailure(this.message);
  final String message;
}

// In BlocBuilder:
switch (state) {
  case AuthInitial() => const SizedBox.shrink(),
  case AuthLoading() => const CircularProgressIndicator(),
  case AuthSuccess(:final user) => UserDashboard(user: user),
  case AuthFailure(:final message) => ErrorBanner(message: message),
};
```

### Rule 5 — Stream Transformers

Custom stream transformations must be expressed as reusable `StreamTransformer` instances rather than inline `map`/`where` chains scattered across the codebase. A transformer encapsulates the transformation logic, can be tested in isolation, and can be composed. Inline chaining is acceptable only for trivial single-step transformations local to a single method.
