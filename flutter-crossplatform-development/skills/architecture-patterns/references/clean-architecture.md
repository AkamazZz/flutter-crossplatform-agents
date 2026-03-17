---
description: >
  Clean Architecture layer separation, dependency rule, and package structure for Flutter projects.
---

# Clean Architecture Reference

## Layer Definitions

### Presentation Layer
Contains everything the user sees and the state machines that drive it.

- **Widgets** — `StatelessWidget` / `StatefulWidget`; pure UI rendering.
- **BLoC / Bloc** — receives domain entities from repositories; emits sealed states consumed by
  widgets. No Flutter SDK imports in BLoC files.
- **Screens** — named widget classes that compose smaller widgets and provide BLoC via
  `BlocProvider`.

### Domain Layer
The core of the application. Has zero dependencies on any framework, library, or other layer.

- **Entities** — plain Dart classes extending `Equatable`; represent business concepts.
- **Repository interfaces** — abstract classes defining contracts that the Data layer must fulfil.
  Return entities or Dart primitives. Annotate with `@useResult`.
- **Use cases / interactors** — optional thin classes that orchestrate one specific business
  operation using injected repository interfaces.
- **Domain exceptions** — typed exceptions (`AuthException`, `NetworkException`) that the rest of
  the app uses. Never wrap `DioException` or other infrastructure exceptions directly.

### Data Layer
Implements the domain contracts. Knows about Flutter, third-party packages, and I/O.

- **Repository implementations** — concrete classes annotated `@immutable` that implement domain
  interfaces. Map DTOs to entities at the boundary before returning.
- **Data sources** — HTTP clients (`Dio`), local storage (`SharedPreferences`, `Hive`), device
  APIs. Each has a single responsibility.
- **DTOs** — `json_serializable` classes; snake_case field names; no business logic. Never
  imported outside the Data layer.
- **Mappers** — static or injectable classes that convert DTO ↔ entity. Use when the mapping is
  bidirectional or requires injected context.

## Dependency Rule

```
Presentation  ──depends on──►  Domain  ◄──depends on──  Data
```

- Domain has **no** imports pointing outward.
- Presentation imports **only** domain interfaces and entities, never Data implementations.
- Data imports domain interfaces to implement them; never imports Presentation.
- Violations in any direction are architectural bugs — flag them in review.

## Package Structure

In a Melos monorepo each layer lives in its own package to enforce the dependency rule at the
Dart import level:

```
packages/
  domain/
    auth/
      lib/
        src/
          entities/
            user.dart
          repositories/
            auth_repository.dart
          exceptions/
            auth_exception.dart
        auth_domain.dart          ← barrel export
  data/
    auth/
      lib/
        src/
          dtos/
            user_dto.dart
          sources/
            auth_remote_source.dart
          repositories/
            auth_repository_impl.dart
        auth_data.dart            ← barrel export
  features/
    auth/
      lib/
        src/
          bloc/
            auth_bloc.dart
            auth_event.dart
            auth_state.dart
          screens/
            login_screen.dart
          widgets/
            login_form.dart
        auth_feature.dart         ← barrel export
```

`pubspec.yaml` dependencies enforce the rule:
- `domain/auth` depends on **nothing** in the monorepo.
- `data/auth` depends on `domain/auth`.
- `features/auth` depends on `domain/auth` only (never on `data/auth`).

## What Goes in Each Layer — Decision Guide

| Artefact | Question | Answer |
|---|---|---|
| New model class | Does it represent a business concept? | Domain entity |
| New model class | Is it a wire format (JSON/protobuf)? | Data DTO |
| New class | Does it call the network or disk? | Data source |
| New class | Does it coordinate data sources + return entities? | Data repository impl |
| New class | Does it define the contract for the above? | Domain repository interface |
| New class | Does it render UI or manage UI state? | Presentation |
| New class | Is it a single business operation with no UI? | Domain use case |

## Cross-Layer Communication

All cross-layer communication is mediated by domain interfaces. The Presentation layer never
knows which concrete class satisfies a repository interface — that wiring happens in
`CompositionRoot` (see `dependency-injection.md`).

```
LoginBloc                  AuthRepository (interface)   AuthRepositoryImpl
  │                               │                            │
  │──── repository.login() ──────►│                            │
  │                               │◄──── implements ───────────│
  │◄─── Result<User, AuthException>                            │
```

DTOs never cross this boundary. `AuthRepositoryImpl.login()` calls the data source, receives an
`AuthResponseDto`, maps it to `User`, and returns `User`.

## Full Feature Structure Example

```
packages/
  domain/
    weather/
      lib/
        src/
          entities/
            weather.dart           # WeatherEntity extends Equatable
            location.dart          # LocationEntity extends Equatable
          repositories/
            weather_repository.dart  # abstract interface
          exceptions/
            weather_exception.dart
        weather_domain.dart

  data/
    weather/
      lib/
        src/
          dtos/
            weather_dto.dart       # @JsonSerializable()
          sources/
            weather_remote_source.dart  # uses Dio
          repositories/
            weather_repository_impl.dart  # @immutable, maps DTO→entity
          mappers/
            weather_mapper.dart
        weather_data.dart

  features/
    weather/
      lib/
        src/
          bloc/
            weather_bloc.dart
            weather_event.dart     # sealed class
            weather_state.dart     # sealed class + Equatable
          screens/
            weather_screen.dart
          widgets/
            temperature_card.dart
            forecast_list.dart
          di/
            weather_dependencies.dart  # WeatherDependencies container
        weather_feature.dart
```

## Anti-Patterns

**Domain depending on Flutter**
```dart
// BAD — entity imports Flutter
import 'package:flutter/material.dart';

class WeatherEntity extends Equatable {
  final Color color; // Flutter type in domain — forbidden
  ...
}
```

**Data layer exposing implementation details**
```dart
// BAD — repository returns DTO instead of entity
abstract class WeatherRepository {
  Future<WeatherDto> getWeather(String city); // DTO in domain interface — forbidden
}
```

**Presentation importing Data directly**
```dart
// BAD — BLoC bypasses domain interface
import 'package:weather_data/weather_data.dart';

class WeatherBloc extends Bloc<...> {
  final WeatherRepositoryImpl _repo; // concrete impl in Presentation — forbidden
}
```

**Correct pattern**
```dart
// GOOD — BLoC depends on domain interface only
import 'package:weather_domain/weather_domain.dart';

class WeatherBloc extends Bloc<WeatherEvent, WeatherState> {
  final WeatherRepository _repository; // abstract interface — correct

  WeatherBloc({required WeatherRepository repository})
      : _repository = repository,
        super(const WeatherState.initial());
}
```
