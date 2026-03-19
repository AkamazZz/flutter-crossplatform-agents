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

- [patterns-records-sealed.md](references/patterns-records-sealed.md) — pattern matching, destructuring, records, sealed classes, guard clauses; consult when modeling states, events, results, or writing exhaustive switches
- [async-isolates.md](references/async-isolates.md) — Future, Stream, StreamController, Isolate.run(), Zone, Completer, stream transformers; consult for any async code, background work, or stream pipelines
- [code-generation.md](references/code-generation.md) — build_runner, json_serializable, freezed, go_router gen, injectable (anti-pattern); consult when setting up code gen or serialization

## Quick Reference

- `switch` expression — `result = switch (x) { pattern => value, ... }` — exhaustive over sealed types, replaces if/else chains
- Record literal — `(name, age)` or `(name: 'Ada', age: 36)` — return multiple values without a class
- Record destructuring — `final (name, age) = record;` — unpack record fields inline
- Object pattern — `case User(:final name, :final age)` — destructure class fields in switch arms
- Guard clause — `case Foo(:final x) when x > 0` — add a boolean condition to a pattern arm
- Sealed class — `sealed class State {}` + subclasses — model closed sets of states/events with exhaustiveness
- `Isolate.run()` — `await Isolate.run(() => heavyWork())` — offload CPU-bound work off the main thread
- Broadcast stream — `StreamController.broadcast()` — allows multiple concurrent listeners
- `Completer` — `final c = Completer<T>(); c.complete(value);` — bridges a callback API into a Future
- `asyncMap` — `stream.asyncMap((e) async => ...)` — async transformation in a stream pipeline

## Type System

- `extension type` — `extension type UserId(String value) implements String` — zero-cost wrappers for typed IDs or units of measure
- Extension methods — `extension on String { ... }` — augment existing types without subclassing
- Bounded generics — `T extends Equatable` — constrain type parameters to required capabilities
- `covariant` override — narrow parameter types in method overrides

**Records vs. classes**: Use records for temporary groupings (`(int count, String label)` return values). Use classes for domain concepts with identity, equality, and methods.

**Exhaustiveness is a feature, not a style choice.** Using `default:` or `_` in a switch on a sealed type defeats the purpose. Handle each case explicitly so the compiler catches missing cases when subtypes are added.

**Streams must be cancelled.** Every `StreamSubscription` must have a corresponding `.cancel()` call. Every `StreamController` must have a `.close()` call. Leaked subscriptions cause memory leaks.

**Code generation is a build step, not a runtime dependency.** The generator never runs at app startup.

## Anti-Patterns

**Do not use nullable fields as a substitute for sealed classes.** A state object with `String? errorMessage` and `bool isLoading` is an uncontrolled product type. Use a sealed class hierarchy instead — every consumer is forced to handle every case at compile time.

**Do not nest switch statements.** If an inner switch is needed, the sealed hierarchy is likely under-specified. Flatten by adding more specific subclasses.

**Do not use `compute()`.** It is deprecated. Use `Isolate.run()` directly.

**Do not use `freezed` for BLoC states or events.** See CLAUDE.md Rule 11.

**Do not use `injectable`.** See CLAUDE.md Rule 13.

