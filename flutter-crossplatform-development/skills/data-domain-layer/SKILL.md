---
name: data-domain-layer
description: >
  Data and Domain layer patterns with entities, DTOs, repositories, API integration, and Drift
  persistence. Use when working with data models, repositories, API clients, local storage,
  mappers, or error handling. Trigger on: entities, DTOs, repositories, API clients, Drift,
  Dio, Retrofit, mappers, JSON serialization, offline, caching.
---

# Data & Domain Layer Skill

This skill enforces the project's Data and Domain layer standards. Before answering, read
the relevant reference file(s) for the topic at hand.

## Architecture Overview

```
BLoC Layer ←→ Domain Layer ←→ Data Layer
               (Entities)       (DTOs / Sources)
               (Interfaces)     (Repository Impl)
```

**Domain Layer** — pure business logic, no infrastructure dependencies.
- Defines **Entities**: plain Dart objects that model the business domain.
- Defines **Repository Interfaces**: contracts that describe *what* data operations exist.

**Data Layer** — infrastructure implementation.
- Implements repository interfaces from the domain.
- Owns DTOs, JSON serialization, API clients, local databases, and caches.
- Maps DTOs → Entities at the repository boundary so the domain stays clean.

Directory structure: see CLAUDE.md — Directory Structure Convention.

## Reference Files — Read Before Answering

| Topic | Reference file | When to read |
|---|---|---|
| Entities, DTOs, repository interfaces | `references/entities-dtos.md` | Defining domain models, DTO classes, repository contracts |
| Mappers, impl, error handling, DI | `references/repository-pattern.md` | Implementing repositories, mapper classes, exception hierarchy, DI setup |
| Drift local database & secure storage | `references/drift-persistence.md` | Local persistence with Drift, SharedPreferences, flutter_secure_storage |
| Dio, Retrofit, interceptors, offline-first | `references/api-integration.md` | HTTP clients, auth interceptors, retry logic, offline queuing |

**Rule**: Always read the relevant reference(s) in full before writing code for that layer.

## Quick Reference

| Concern | Rule |
|---|---|
| Entities | Pure Dart, `Equatable`, no JSON, no Flutter imports, `copyWith`, business validation allowed |
| DTOs | `json_serializable` + `FieldRename.snake`, no business logic, never the same class as the entity |
| Repository interface | `abstract interface class`, returns Entities/primitives, `@useResult` on data-returning methods |
| Repository impl | `@immutable`, constructor-inject all sources and mappers, map at boundary, cache in impl |
| Mappers | Use for bidirectional or complex mapping; inject sub-mappers via constructor |
| Exceptions | `DomainException` hierarchy; catch `DioException` in impl and rethrow as domain exception |
| BLoC exposure | **Never expose DTOs to BLoC** — always map to entities first |

## Anti-Patterns

1. Passing a DTO directly to a BLoC event or state — always map first.
2. Adding JSON annotations (`@JsonSerializable`) to an Entity — entities must be pure Dart.
3. Importing `package:dio` or `package:retrofit` inside the domain layer — domain is infra-free.
4. Business logic inside a DTO or mapper — belongs in the Entity or a domain service.
5. Creating a repository instance inside a BLoC — use constructor injection.
6. Catching `DioException` in the BLoC — the repository must absorb and remap it.
7. Using Hive for local storage — use Drift or SharedPreferences (Rule 6).
8. Mutable fields on entities or repository impls — all data classes must be immutable.
