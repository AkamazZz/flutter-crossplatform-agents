---
name: bloc-state-management
description: >
  BLoC state management patterns with sealed classes, event transformers, and race condition
  prevention. Use when writing BLoC classes, events, states, transformers, or debugging state
  management issues. Trigger on: BLoC, Cubit, events, states, transformers, race conditions,
  state management, emit, sequential, droppable, restartable.
---

# BLoC State Management

This skill enforces BLoC architecture standards for state management across all BLoC layer
concerns. Before answering, read the relevant reference file(s) for the topic at hand.

## Architecture Overview

```
Widget Layer ↔ BLoC Layer ↔ Domain Layer ↔ Data Layer
(UI/View)      (Business Logic)  (Entities/Interfaces)  (Repositories/Sources)
```

## Reference Files — Read Before Answering

| Topic | Reference file | When to read |
|---|---|---|
| Core BLoC rules | `references/bloc-core-rules.md` | Writing/reviewing any BLoC class, understanding architecture layers, file structure, anti-patterns |
| Event design | `references/event-design.md` | Writing events, naming events, grouping events, using State Emitter mixin |
| State design | `references/state-design.md` | Writing state hierarchies, Equatable props, state copy patterns, convenience getters |
| Transformer strategies | `references/transformer-strategies.md` | Choosing sequential/droppable/restartable, stream subscriptions, preventing race conditions |

**Rule**: Always read the relevant reference(s) in full before writing code for that layer.

## Quick Reference by Layer

| Concern | Rule |
|---|---|
| State shape | Sealed class + Equatable, immutable, no logic |
| Event shape | Sealed class, action-oriented names (Load not Loaded) |
| Transformers | Always explicit; default = `sequential()` |
| Repositories | Constructor injection only, never `getIt` inside BLoC |
| BLoC↔BLoC | Never — share via repository streams using `emit.forEach`/`emit.onEach` |
| Cubit | Never, except streams or learning phase |
| Public methods | None — only `add()`, `stream`, `state` |
| Flutter imports | Not allowed in BLoC layer (except `foundation`) |

## Key Anti-Patterns to Flag Immediately

1. Using `Cubit` instead of `Bloc` — no event processing order, race condition risk
2. Creating repository inside BLoC (`final _repo = MyRepo()`) — breaks constructor injection rule
3. Bare `on<Event>(handler)` without explicit transformer — silent race condition
4. BLoC depending on another BLoC via constructor — use repository streams instead
5. Mutable state fields — any non-`final` field in a state class is forbidden
6. Emitting the same state object by reference — always create a new instance
