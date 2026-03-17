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

| Topic | Reference file | When to read |
|---|---|---|
| Clean Architecture | `references/clean-architecture.md` | Layer separation, package structure, cross-layer communication, what belongs where |
| Feature-Driven Development | `references/feature-driven-development.md` | Feature-first organization, shared kernel, Melos monorepo, feature containers |
| Dependency Injection | `references/dependency-injection.md` | CompositionRoot, AppScope, constructor injection, DI vs service locator |

**Rule**: Always read the relevant reference(s) in full before writing or reviewing architectural
code. Multiple references may apply — a new feature setup needs all three.

## Quick Reference by Layer

### Presentation Layer

| Concern | Rule |
|---|---|
| Widgets | Stateless by default; no business logic |
| BLoC | Constructor-injected repositories only; no service locator |
| Navigation | Defined at feature boundary; no cross-feature widget imports |
| DI access | `context.di` / `context.featureXDependencies` extensions |

### Domain Layer

| Concern | Rule |
|---|---|
| Entities | Pure Dart, Equatable, zero external dependencies |
| Repository interfaces | Abstract classes only; return entities or primitives |
| Use cases / interactors | Optional; pure Dart, injected repositories |
| Forbidden imports | Flutter SDK, Dio, json_serializable, Data layer |

### Data Layer

| Concern | Rule |
|---|---|
| Repository impl | Implements domain interface; maps DTOs to entities at boundary |
| DTOs | `json_serializable`; never exposed to BLoC or domain |
| Data sources | Injected via constructor; one responsibility each |
| Exceptions | Translate to domain exceptions at the repository boundary |

## Global Rules Embedded in This Skill

**Global Rule 2 — Constructor injection only**
All dependencies must be provided through constructors. Service locators (`GetIt`, `injectable`,
`ComponentHolder`, any global singleton registry) are forbidden everywhere in the codebase.
Flag any use of `getIt<>()`, `locator<>()`, or `sl<>()` immediately.

**Global Rule 3 — No DTO exposure to BLoC**
DTOs are a Data layer implementation detail. Repository implementations must map DTOs to domain
entities before returning. BLoC and UI code must never import or reference a DTO class.

## Anti-Patterns — Flag Immediately

1. **Circular dependencies** between packages or layers (e.g., domain importing data).
2. **Domain importing Flutter** — entities or interfaces importing `package:flutter/*`.
3. **BLoC knowing about widgets** — BLoC importing widget classes or BuildContext.
4. **Repository creating its own dependencies** — `final _client = Dio()` inside a repo.
5. **DTO leaking into BLoC** — passing a raw DTO to `add(SomeEvent(dto: ...))`.
6. **Service locator inside a class** — `GetIt.instance<Foo>()` called inside a method body.
