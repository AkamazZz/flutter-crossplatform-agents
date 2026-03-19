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

- `references/bloc-core-rules.md` — BLoC class structure, architecture layers, file layout, anti-patterns; read before writing or reviewing any BLoC
- `references/event-design.md` — event naming, grouping, State Emitter mixin; read when designing event hierarchies
- `references/state-design.md` — sealed state hierarchies, Equatable props, copy patterns, convenience getters
- `references/transformer-strategies.md` — sequential/droppable/restartable selection, stream subscriptions, race condition prevention

**Rule**: Always read the relevant reference(s) in full before writing code for that layer.

## Quick Reference by Layer

- State shape — sealed class + Equatable, immutable, no logic
- Event shape — sealed class, action-oriented names (Load not Loaded)
- Transformers — always explicit; default = `sequential()`
- Repositories — constructor injection only; never `getIt` inside BLoC
- BLoC↔BLoC — never; share via repository streams using `emit.forEach`/`emit.onEach`
- Cubit — never, except pure stream forwarding or explicit learning context
- Public methods — none; only `add()`, `stream`, `state`
- Flutter imports — not allowed in BLoC layer except `foundation`

## Key Anti-Patterns to Flag Immediately

1. Using `Cubit` instead of `Bloc` — no event processing order, race condition risk
2. Creating repository inside BLoC (`final _repo = MyRepo()`) — breaks constructor injection rule
3. Bare `on<Event>(handler)` without explicit transformer — silent race condition
4. BLoC depending on another BLoC via constructor — use repository streams instead
5. Mutable state fields — any non-`final` field in a state class is forbidden
6. Emitting the same state object by reference — always create a new instance
