---
description: >
  Feature-driven directory organization, feature isolation, shared kernel patterns, and monorepo with Melos.
---

# Feature-Driven Development Reference

## Feature-First vs Layer-First

### Layer-first organization
```
lib/
  blocs/
    auth_bloc.dart
    weather_bloc.dart
  screens/
    login_screen.dart
    weather_screen.dart
  repositories/
    auth_repository.dart
    weather_repository.dart
```

Problems at scale: adding a new feature requires touching every top-level folder; unrelated files
sit next to each other; impossible to enforce feature boundaries; cross-feature coupling grows
silently.

### Feature-first organization
```
packages/
  features/
    auth/       ← self-contained: bloc, screens, widgets, DI container
    weather/    ← self-contained: bloc, screens, widgets, DI container
    profile/
  domain/
    auth/       ← contract owned by auth feature
    weather/
  data/
    auth/       ← implementation owned by auth feature
    weather/
  shared/
    ui_kit/     ← shared kernel: theme, common widgets
    core/       ← base classes, extensions, logging
```

Why feature-first wins at scale:
- A feature is a deployable / testable unit. Its blast radius is bounded.
- Teams can own features without stepping on each other.
- Adding a feature means adding a package, not editing five existing folders.
- Dependency rule is enforced by the package graph, not by convention.

## Feature Isolation

Each feature package is self-contained. It owns:

- BLoC + events + states → `features/[name]/lib/src/bloc/`
- Screens → `features/[name]/lib/src/screens/`
- Feature-private widgets → `features/[name]/lib/src/widgets/`
- DI container class → `features/[name]/lib/src/di/[name]_dependencies.dart`
- Route definitions → `features/[name]/lib/src/routes/`
- Barrel export → `features/[name]/lib/[name]_feature.dart`

Rules for isolation:
- Feature A must **never** import widgets, BLoC, or screens from Feature B.
- Features communicate through the domain layer only (shared entities, callbacks, streams from
  repositories).
- Navigation between features is coordinated at the app shell level — features expose named
  routes or callbacks; they do not push into each other's route stacks.

## Shared Kernel

Code that is genuinely cross-feature belongs in a shared package, not copied into each feature.

```
packages/
  shared/
    ui_kit/
      lib/
        src/
          theme/
            app_theme.dart
            app_colors.dart
            app_text_styles.dart
          widgets/
            app_button.dart
            app_text_field.dart
            loading_indicator.dart
          extensions/
            context_extensions.dart
        ui_kit.dart
    core/
      lib/
        src/
          base/
            base_bloc.dart       ← optional; shared BLoC base if needed
          logging/
            logger.dart
          result/
            result.dart          ← Result<T, E> type
          config/
            app_config.dart
        core.dart
```

Rule: the shared kernel must not depend on any feature package. If you find yourself wanting to
import a feature into `ui_kit`, extract the shared concept into a domain entity instead.

## Monorepo with Melos

### Workspace setup (`melos.yaml`)

```yaml
name: my_flutter_app
packages:
  - packages/**

scripts:
  build:
    run: melos exec -- dart run build_runner build --delete-conflicting-outputs
    description: Run build_runner in all packages
    packageFilters:
      dependsOn: build_runner

  test:
    run: melos exec -- flutter test
    description: Run tests in all packages
    packageFilters:
      flutter: true

  analyze:
    run: melos exec -- flutter analyze
    description: Analyze all packages

  clean:
    run: melos exec -- flutter clean
    description: Clean all packages

  bootstrap:
    run: melos bootstrap
    description: Link local packages and fetch dependencies
```

### Package pubspec dependencies

`packages/features/auth/pubspec.yaml`:
```yaml
name: auth_feature
dependencies:
  auth_domain:
    path: ../../domain/auth
  ui_kit:
    path: ../../shared/ui_kit
  core:
    path: ../../shared/core
  flutter_bloc: ^8.0.0
  provider: ^6.0.0
```

`packages/data/auth/pubspec.yaml`:
```yaml
name: auth_data
dependencies:
  auth_domain:
    path: ../../domain/auth
  dio: ^5.0.0
  json_annotation: ^4.0.0
dev_dependencies:
  json_serializable: ^6.0.0
  build_runner: ^2.0.0
```

`packages/domain/auth/pubspec.yaml`:
```yaml
name: auth_domain
dependencies:
  equatable: ^2.0.0
  meta: ^1.0.0
  # Nothing else — no Flutter, no Dio
```

### Common Melos commands

```bash
melos bootstrap          # link all local packages, run pub get
melos run build          # generate code across all packages
melos run test           # run all tests
melos run analyze        # static analysis across all packages
melos list               # list all managed packages
melos exec -- dart pub outdated  # check outdated deps in all packages
```

## Feature Container Pattern for DI

Each feature owns a `Dependencies` container and a `DependenciesBuilder`. See
`dependency-injection.md` for the full wiring pattern. The feature container holds only the
dependencies that belong to that feature:

```dart
// packages/features/weather/lib/src/di/weather_dependencies.dart

class WeatherDependencies {
  const WeatherDependencies({
    required this.weatherRepository,
  });

  final WeatherRepository weatherRepository;
}

abstract class WeatherDependenciesBuilder {
  static WeatherDependencies build(CoreDependencies core) {
    final remoteSource = WeatherRemoteSource(dio: core.dio);
    final repository = WeatherRepositoryImpl(remoteSource: remoteSource);

    return WeatherDependencies(weatherRepository: repository);
  }
}
```

The feature container is then registered in `AppScope` via `MultiProvider` and accessed through a
context extension:

```dart
extension WeatherDependenciesX on BuildContext {
  WeatherDependencies get weatherDependencies =>
      read<WeatherDependencies>();
}
```

## When to Split a Feature into Sub-Features

Split when any of the following are true:

- Feature has 5+ screens covering distinct user journeys → split by journey (e.g., `auth_login`, `auth_registration`)
- Feature has 3+ independent BLoCs with no shared state → split into sub-features sharing a domain package
- Two teams need to work on the feature simultaneously → split to eliminate merge conflicts
- Feature's `pubspec.yaml` has 10+ local path dependencies → decompose, it is doing too much
- Need to reuse one sub-part of the feature elsewhere → extract that sub-part as a separate feature or domain package

Do **not** split prematurely. Start with a single feature package and extract only when one of
the signals above appears. Over-splitting creates coordination overhead without benefit.
