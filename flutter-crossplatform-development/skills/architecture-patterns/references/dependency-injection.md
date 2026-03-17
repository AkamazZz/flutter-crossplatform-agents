---
description: >
  Pure DI with CompositionRoot, AppScope, constructor injection patterns, and anti-patterns to avoid.
---

# Dependency Injection Reference

## GLOBAL RULE 2 — Constructor Injection Only

> All dependencies must be provided through constructors. Service locators (`GetIt`, `injectable`,
> `ComponentHolder`, any global singleton registry) are **forbidden** everywhere in the codebase.

Flag any occurrence of `getIt<>()`, `locator<>()`, `sl<>()`, `GetIt.instance`, or
`ServiceLocator.of()` immediately in code review. This is a non-negotiable architectural rule.

## Pure DI Principle

The project uses Pure DI — manually composing and injecting dependencies — combined with
Flutter's `InheritedWidget` / `Provider` for propagating the dependency graph through the
widget tree.

Pure DI means:

```dart
// Explicit, readable, testable — no magic
final databaseConnection = DatabaseConnection.open();
final userDatasource = UserDatasource(conn: databaseConnection);
final userRepository = UserRepository(datasource: userDatasource);
final userBloc = UserBloc(repository: userRepository);
```

Every dependency is visible at the construction site. There are no hidden look-ups, no global
registries, no implicit resolution.

## Dependencies Container

Define a central `Dependencies` class as a container for all long-lived objects. The class itself
is pure Dart — easy to instantiate in tests.

```dart
// lib/src/di/dependencies.dart

class Dependencies {
  const Dependencies({
    required this.core,
    required this.auth,
    required this.weather,
  });

  final CoreDependencies core;
  final AuthDependencies auth;
  final WeatherDependencies weather;
}
```

## Feature Containers

Dependencies that belong to the same feature are grouped in a feature container instead of
cluttering the top-level `Dependencies` class:

```dart
// packages/features/auth/lib/src/di/auth_dependencies.dart

class AuthDependencies {
  const AuthDependencies({
    required this.authRepository,
    required this.secureStorage,
  });

  /// Repository used by AuthBloc for sign-in / sign-out operations.
  final AuthRepository authRepository;

  /// Secure storage used to persist tokens.
  final SecureStorage secureStorage;
}

// packages/features/weather/lib/src/di/weather_dependencies.dart

class WeatherDependencies {
  const WeatherDependencies({
    required this.weatherRepository,
  });

  final WeatherRepository weatherRepository;
}
```

## CoreDependenciesBuilder / FeatureDependenciesBuilder

Each dependency group has a corresponding builder that wires concrete implementations to domain
interfaces. Builders are abstract classes with a single static `build` method.

```dart
abstract class CoreDependenciesBuilder {
  static Future<CoreDependencies> build(AppConfig config) async {
    final sharedPrefs = await SharedPreferences.getInstance();
    final dio = Dio(BaseOptions(baseUrl: config.baseUrl));

    return CoreDependencies(
      dio: dio,
      sharedPreferences: sharedPrefs,
      config: config,
    );
  }
}

abstract class AuthDependenciesBuilder {
  static AuthDependencies build(CoreDependencies core) {
    final secureStorage = SecureStorageImpl(prefs: core.sharedPreferences);
    final repository = AuthRepositoryImpl(
      networkClient: core.dio,
      storage: secureStorage,
    );

    return AuthDependencies(
      authRepository: repository,
      secureStorage: secureStorage,
    );
  }
}

abstract class WeatherDependenciesBuilder {
  static WeatherDependencies build(CoreDependencies core) {
    final remoteSource = WeatherRemoteSource(dio: core.dio);
    final repository = WeatherRepositoryImpl(remoteSource: remoteSource);

    return WeatherDependencies(weatherRepository: repository);
  }
}
```

## CompositionRoot

`CompositionRoot` is the **single place** where the entire dependency graph is assembled.
Nothing outside `CompositionRoot` constructs concrete implementations.

```dart
final class CompositionRoot {
  const CompositionRoot(this.config, this.logger);

  final AppConfig config;
  final ILogger logger;

  /// Assembles the full dependency graph.
  /// Called once from [AppRunner] before [runApp].
  Future<Dependencies> initDependencies() async {
    try {
      final stopwatch = Stopwatch()..start();
      logger.info('Initialization started...');

      // 1. Initialize infrastructure (DB, network)
      final core = await CoreDependenciesBuilder.build(config);

      // 2. Initialize feature groups, passing core to each
      final auth = AuthDependenciesBuilder.build(core);
      final weather = WeatherDependenciesBuilder.build(core);

      // 3. If Feature B depends on Feature A, pass A to B's builder:
      // final profile = ProfileDependenciesBuilder.build(core, auth);

      stopwatch.stop();
      logger.info(
        'Dependencies initialized in ${stopwatch.elapsedMilliseconds}ms',
      );

      return Dependencies(
        core: core,
        auth: auth,
        weather: weather,
      );
    } catch (e, stackTrace) {
      logger.error('Failed to initialize dependencies', e, stackTrace);
      rethrow;
    }
  }
}
```

## AppRunner Initialization

`AppRunner` runs before `runApp`. It initializes Flutter bindings, sets up global error handlers
and BLoC observers, then delegates to `CompositionRoot`.

```dart
/// Responsible for initialization and running the app.
final class AppRunner {
  const AppRunner();

  Future<void> initializeAndRun() async {
    final binding = WidgetsFlutterBinding.ensureInitialized();

    // Preserve the splash screen while we initialize.
    binding.deferFirstFrame();

    // Global error handlers.
    FlutterError.onError = logger.logFlutterError;
    WidgetsBinding.instance.platformDispatcher.onError =
        logger.logPlatformDispatcherError;

    // BLoC configuration.
    Bloc.observer = AppBlocObserver(logger);
    Bloc.transformer = bloc_concurrency.sequential();

    const config = AppConfig();
    final dependencies =
        await CompositionRoot(config, logger).initDependencies();

    // Allow rendering now that dependencies are ready.
    binding.allowFirstFrame();

    runApp(App(dependencies: dependencies));
  }
}
```

Entry point:

```dart
// lib/main.dart
void main() => const AppRunner().initializeAndRun();
```

## AppScope

`AppScope` exposes the assembled `Dependencies` to the entire widget tree via `MultiProvider`.
Wrap the root `MaterialApp` (or `App`) in `AppScope`.

```dart
class AppScope extends StatelessWidget {
  const AppScope({
    required this.dependencies,
    required this.child,
    super.key,
  });

  final Dependencies dependencies;
  final Widget child;

  @override
  Widget build(BuildContext context) {
    return MultiProvider(
      providers: [
        // Register each feature container separately.
        Provider<CoreDependencies>.value(value: dependencies.core),
        Provider<AuthDependencies>.value(value: dependencies.auth),
        Provider<WeatherDependencies>.value(value: dependencies.weather),
      ],
      child: child,
    );
  }
}
```

Usage in `App`:

```dart
class App extends StatelessWidget {
  const App({required this.dependencies, super.key});

  final Dependencies dependencies;

  @override
  Widget build(BuildContext context) {
    return AppScope(
      dependencies: dependencies,
      child: MaterialApp.router(
        routerConfig: AppRouter.config(),
      ),
    );
  }
}
```

## context.di / context.featureXDependencies Extensions

Access dependencies in the presentation layer through typed context extensions. Never call
`context.read<Dependencies>()` directly in widget or BLoC code — use the extensions.

```dart
// packages/shared/core/lib/src/di/di_extensions.dart

extension DiExtensions on BuildContext {
  CoreDependencies get di => read<CoreDependencies>();
}

// packages/features/weather/lib/src/di/weather_di_extensions.dart

extension WeatherDiExtensions on BuildContext {
  WeatherDependencies get weatherDependencies =>
      read<WeatherDependencies>();
}

// packages/features/auth/lib/src/di/auth_di_extensions.dart

extension AuthDiExtensions on BuildContext {
  AuthDependencies get authDependencies => read<AuthDependencies>();
}
```

Usage in a screen when providing a BLoC:

```dart
class WeatherScreen extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return BlocProvider(
      create: (_) => WeatherBloc(
        repository: context.weatherDependencies.weatherRepository,
      ),
      child: const WeatherView(),
    );
  }
}
```

## DI vs Service Locator — Why Service Locator is Forbidden

| Concern | Pure DI (this project) | Service Locator (forbidden) |
|---|---|---|
| Dependency visibility | Explicit — declared in constructor | Hidden — looked up inside method bodies |
| Testability | Mock by passing a different constructor argument | Requires re-registering globals before each test |
| Coupling | Loose — classes don't know where deps come from | Tight — classes depend on the global registry |
| Inversion of Control | Respected — deps are pushed in | Violated — classes pull deps out |
| Compile-time safety | Yes — missing deps are compile errors | No — missing registrations fail at runtime |

Common service locator patterns that are **forbidden**:

```dart
// FORBIDDEN — GetIt
final weatherRepo = GetIt.instance<WeatherRepository>();

// FORBIDDEN — injectable generated locator
@injectable
class WeatherBloc { ... }

// FORBIDDEN — custom ComponentHolder
WeatherBloc(repository: ComponentHolder.weatherFeature.weatherRepository);

// FORBIDDEN — any global singleton access pattern
final repo = ServiceLocator.of<WeatherRepository>();
```

The correct pattern is always constructor injection wired at `CompositionRoot`:

```dart
// CORRECT
final weatherBloc = WeatherBloc(
  repository: context.weatherDependencies.weatherRepository,
);
```
