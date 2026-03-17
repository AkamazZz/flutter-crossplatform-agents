---
description: >
  Code generation with build_runner, json_serializable, freezed, and custom builders.
---

# Code Generation

Code generation in Dart uses `build_runner` to transform annotated source files into `.g.dart` or `.freezed.dart` output files at build time. Generated files are checked into source control.

---

## build_runner Setup

### pubspec.yaml

```yaml
dependencies:
  json_annotation: ^4.9.0
  freezed_annotation: ^2.4.0   # only if using freezed

dev_dependencies:
  build_runner: ^2.4.0
  json_serializable: ^6.7.0
  freezed: ^2.4.0              # only if using freezed
```

### build.yaml (optional)

Place `build.yaml` at the project root to configure builder options globally.

```yaml
targets:
  $default:
    builders:
      json_serializable:
        options:
          explicit_to_json: true
          field_rename: snake
```

`explicit_to_json: true` forces nested objects to call their own `toJson()` rather than being cast. Always enable this to avoid silent serialization bugs with nested models.

### Running the Generator

```bash
dart run build_runner build --delete-conflicting-outputs
```

Use `--delete-conflicting-outputs` to remove stale generated files that conflict with new output. Without it, build_runner errors when an output file already exists with different content.

For continuous generation during development:

```bash
dart run build_runner watch --delete-conflicting-outputs
```

---

## json_serializable

`json_serializable` generates `fromJson` factory constructors and `toJson` methods for model classes.

### Basic Usage

```dart
import 'package:json_annotation/json_annotation.dart';

part 'user.g.dart';

@JsonSerializable()
class User {
  const User({required this.id, required this.email, required this.displayName});

  factory User.fromJson(Map<String, dynamic> json) => _$UserFromJson(json);

  final String id;
  final String email;
  @JsonKey(name: 'display_name')
  final String displayName;

  Map<String, dynamic> toJson() => _$UserToJson(this);
}
```

### @JsonKey

`@JsonKey` overrides per-field serialization behavior.

```dart
@JsonKey(name: 'created_at')
final DateTime createdAt;

@JsonKey(includeFromJson: false, includeToJson: false)
final String transientField;

@JsonKey(defaultValue: false)
final bool isVerified;
```

### FieldRename

Set globally in `build.yaml` (preferred) or per-class:

```dart
@JsonSerializable(fieldRename: FieldRename.snake)
class UserProfile { ... }
```

Options: `FieldRename.none` (default), `FieldRename.snake`, `FieldRename.kebab`, `FieldRename.pascal`.

### Nested Objects

```dart
@JsonSerializable(explicitToJson: true)
class Order {
  factory Order.fromJson(Map<String, dynamic> json) => _$OrderFromJson(json);

  final String id;
  final User user; // User must also have fromJson/toJson
  final List<LineItem> items;

  Map<String, dynamic> toJson() => _$OrderToJson(this);
}
```

With `explicitToJson: true`, `user.toJson()` is called rather than casting `user` to `Map`. Without it, nested objects are not serialized correctly.

---

## freezed

freezed generates immutable value classes with `==`, `hashCode`, `copyWith`, and optionally union types.

**Status: ALTERNATIVE — acceptable for complex DTOs and value objects outside the BLoC layer. NOT recommended for BLoC states or events.**

BLoC states and events are modeled as sealed class hierarchies (Global Rule 4). Introducing freezed into the BLoC layer adds a build-time dependency to the core domain, generates a large amount of output code, and provides no benefit over hand-written sealed classes with `Equatable`. Freezed's union syntax also diverges from Dart's native sealed/pattern syntax, reducing readability.

Freezed is acceptable when:
- A DTO or value object has many optional fields and copyWith is genuinely useful.
- The class lives in the data or infrastructure layer, not the domain/BLoC layer.
- The team has explicitly agreed to use it for a specific class.

### Basic Usage (when acceptable)

```dart
import 'package:freezed_annotation/freezed_annotation.dart';

part 'address.freezed.dart';
part 'address.g.dart'; // if also using json_serializable

@freezed
class Address with _$Address {
  const factory Address({
    required String street,
    required String city,
    @Default('') String state,
    required String postalCode,
  }) = _Address;

  factory Address.fromJson(Map<String, dynamic> json) => _$AddressFromJson(json);
}
```

### What freezed generates

- `==` and `hashCode` based on all fields.
- `copyWith` with named optional parameters for every field.
- `toString`.
- `fromJson`/`toJson` when combined with `@JsonSerializable`.

### What freezed does NOT replace

- Sealed class exhaustiveness. Freezed's union type (`@Freezed() ... = _StateA; ... = _StateB;`) is a separate code-gen pattern and does not produce a Dart `sealed` class. Exhaustiveness is enforced differently and the compiler cannot verify it in the same way.

---

## go_router Code Generation

go_router supports type-safe route generation via annotations. Cross-reference: `navigation.md`.

```dart
part 'router.g.dart';

@TypedGoRoute<HomeRoute>(path: '/')
class HomeRoute extends GoRouteData {
  const HomeRoute();
}

@TypedGoRoute<UserRoute>(path: '/users/:id')
class UserRoute extends GoRouteData {
  const UserRoute({required this.id});
  final String id;
}
```

Run `build_runner` as normal. The generated file provides `HomeRoute().go(context)` and `UserRoute(id: '123').push(context)`.

---

## injectable — ANTI-PATTERN

injectable is a code generation package that automates registration with the `get_it` service locator. It is documented here only to flag it as prohibited.

**Do not use injectable in this project.**

Reason: injectable is built on `get_it`, a global service locator. Using a service locator violates Global Rule 2, which requires constructor injection throughout the application. Service locators hide dependencies, make unit testing harder (you must configure the global locator in every test), and couple modules to a global registry.

The correct dependency injection approach is explicit constructor injection, with dependencies wired at composition roots (main.dart or feature-level provider trees). No code generation is required for this.

If you see `@injectable`, `@singleton`, `@lazySingleton`, or `GetIt.instance` in a code review, flag it as a violation of Global Rule 2.

---

## Custom Builder Basics

A custom builder is a Dart package that reads source files and emits generated files.

### Minimal Structure

```
my_builder/
  lib/
    builder.dart       # exports the Builder
  build.yaml           # declares the builder
  pubspec.yaml
```

```dart
// builder.dart
import 'package:build/build.dart';

Builder myBuilder(BuilderOptions options) => MyBuilder();

class MyBuilder implements Builder {
  @override
  Map<String, List<String>> get buildExtensions => {
    '.dart': ['.my_builder.dart'],
  };

  @override
  Future<void> build(BuildStep buildStep) async {
    final inputId = buildStep.inputId;
    final source = await buildStep.readAsString(inputId);
    // analyze source, generate output
    final outputId = inputId.changeExtension('.my_builder.dart');
    await buildStep.writeAsString(outputId, generatedContent);
  }
}
```

```yaml
# build.yaml (in the builder package)
builders:
  my_builder:
    import: 'package:my_builder/builder.dart'
    builder_factories: ['myBuilder']
    build_extensions: {'.dart': ['.my_builder.dart']}
    auto_apply: dependents
    build_to: source
```

Custom builders are rarely needed in application code. They are appropriate for internal infrastructure packages that generate boilerplate across many consumers.

---

## Common Mistakes

**Not running build_runner after modifying annotated files.** Generated files go stale silently. The app will fail to compile.

**Committing generated files with local path artifacts.** Generated files must be reproducible from source. Do not edit `.g.dart` or `.freezed.dart` files manually.

**Using freezed for BLoC states or events.** See freezed section above.

**Using injectable.** See injectable section above.

**Omitting `--delete-conflicting-outputs`.** Without it, build_runner will fail on the first conflicting output after a rename or restructure.
