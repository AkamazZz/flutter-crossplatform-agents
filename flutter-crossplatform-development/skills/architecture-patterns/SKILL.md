---
name: architecture-patterns
description: >
  Flutter Clean Architecture, feature-driven development, and dependency injection patterns.
  Use when architecting Flutter apps, organizing features, setting up DI, or defining module
  boundaries. Trigger on: Clean Architecture, project structure, feature organization, DI,
  dependency injection, CompositionRoot, AppScope, packages, modules, Melos.
---

# Architecture Patterns Skill

This skill enforces the project's structural and architectural standards across layer separation,
feature organization, and dependency wiring. Before answering, read the relevant reference file(s)
for the topic at hand.

## Architecture Overview

Three-layer architecture with a strict unidirectional dependency rule:

```
Presentation Layer  →  Domain Layer  →  Data Layer
(Widgets + BLoC)       (Entities +       (Repo impls +
                        Repo interfaces)   Sources + DTOs)
```

**Dependency rule**: inner layers never import outer layers. Domain knows nothing about Flutter,
Data, or BLoC. Data depends on Domain interfaces. Presentation depends on Domain interfaces only —
never on concrete Data implementations.

## Reference Files — Read Before Answering

- `references/clean-architecture.md` — layer separation, package structure, cross-layer communication; consult for what belongs in each layer
- `references/feature-driven-development.md` — feature-first organization, shared kernel, Melos monorepo, feature containers
- `references/dependency-injection.md` — CompositionRoot, AppScope, constructor injection, DI vs service locator

**Rule**: Always read the relevant reference(s) in full before writing or reviewing architectural
code. Multiple references may apply — a new feature setup needs all three.

## Quick Reference by Layer

### Presentation Layer

- Widgets — stateless by default; no business logic
- BLoC — constructor-injected repositories only; no service locator
- Navigation — defined at feature boundary; no cross-feature widget imports
- DI access — `context.di` / `context.featureXDependencies` extensions

### Domain Layer

- Entities — pure Dart, Equatable, zero external dependencies
- Repository interfaces — abstract classes only; return entities or primitives
- Use cases / interactors — optional; pure Dart, injected repositories
- Forbidden imports — Flutter SDK, Dio, json_serializable, Data layer

### Data Layer

- Repository impl — implements domain interface; maps DTOs to entities at boundary
- DTOs — `json_serializable`; never exposed to BLoC or domain
- Data sources — injected via constructor; one responsibility each
- Exceptions — translate to domain exceptions at the repository boundary

## Anti-Patterns — Flag Immediately

1. **Circular dependencies** between packages or layers (e.g., domain importing data).
2. **Domain importing Flutter** — entities or interfaces importing `package:flutter/*`.
3. **BLoC knowing about widgets** — BLoC importing widget classes or BuildContext.
4. **Repository creating its own dependencies** — `final _client = Dio()` inside a repo.
5. **DTO leaking into BLoC** — passing a raw DTO to `add(SomeEvent(dto: ...))`.
6. **Service locator inside a class** — `GetIt.instance<Foo>()` called inside a method body.
