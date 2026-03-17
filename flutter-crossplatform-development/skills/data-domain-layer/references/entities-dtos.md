---
description: Entities, DTOs, and repository interface rules for the Domain and Data layers.
---

# Entities, DTOs, and Repository Interfaces

## 1. Layer Overview

**Domain Layer** — pure business logic, independent of infrastructure.
- **Entities**: plain Dart objects that reflect the business domain.
- **Repository Interfaces**: contracts that define *what* data operations exist, not *how*.

**Data Layer** — infrastructure implementation.
- Implements repository interfaces defined in the domain.
- Owns DTOs and all JSON/serialization concerns.
- Maps DTOs to Domain Entities before returning from any repository method.

The boundary between layers is enforced by the mapper: DTOs never cross into the domain,
entities never carry JSON annotations.

---

## 2. Directory Structure

```
packages/
  domain/[feature]/lib/
    entities/[feature]_entity.dart        ← pure Dart entity
    repositories/[feature]_repository.dart ← abstract interface

  data/[feature]/lib/
    repositories/[feature]_repository_impl.dart
    sources/
      remote/[feature]_api_client.dart
      local/[feature]_local_storage.dart
    models/[feature]_dto.dart             ← JSON DTO
    mappers/[feature]_mapper.dart
```

---

## 3. Domain Layer: Entities

### Rules

- Pure Dart objects — no infrastructure dependencies.
- Extend `Equatable` for structural (value) equality.
- Provide a `copyWith` method for immutable updates.
- May contain business validation logic (e.g., `isEmailValid()`).
- **No** JSON serialization — that belongs in DTOs.
- **No** Flutter imports (except truly foundational types if unavoidable).

### Full Example

```dart
import 'package:equatable/equatable.dart';

class UserProfile extends Equatable {
  final String id;
  final String name;
  final String email;

  const UserProfile({
    required this.id,
    required this.name,
    required this.email,
  });

  /// Business validation — belongs here, not in a DTO.
  bool isEmailValid() => email.contains('@');

  UserProfile copyWith({String? id, String? name, String? email}) {
    return UserProfile(
      id: id ?? this.id,
      name: name ?? this.name,
      email: email ?? this.email,
    );
  }

  @override
  List<Object?> get props => [id, name, email];
}
```

### Key Points

| Rule | Detail |
|---|---|
| `Equatable` | Always extend it — never override `==` manually |
| `copyWith` | Required on every entity so callers can produce modified copies |
| Business logic | Light validation methods are allowed and encouraged |
| No `@JsonSerializable` | Adding JSON annotations to an entity is a bug |

---

## 4. Domain Layer: Repository Interfaces

### Rules

- `abstract interface class` — defines *what*, not *how*.
- All methods return Domain Entities or primitives, never DTOs.
- Use `@useResult` on every method that returns data so callers are warned if they discard it.
- No implementation details, no infrastructure imports.

### Full Example

```dart
import 'package:meta/meta.dart';

abstract interface class IProfileRepository {
  @useResult
  Future<UserProfile> getProfile();

  Future<void> updateProfile(UserProfile profile);

  @useResult
  Stream<UserProfile> watchProfile();

  Future<void> deleteProfile(String userId);
}
```

### Key Points

| Rule | Detail |
|---|---|
| `abstract interface class` | Prefer `interface` keyword over plain `abstract class` |
| Return types | Only entities and primitives (`String`, `bool`, `void`, `Stream<Entity>`) |
| `@useResult` | Annotate every method whose return value callers must not ignore |
| No `DioException` | The interface must be infra-agnostic — exceptions are domain exceptions |

---

## 5. Data Layer: DTOs (Data Transfer Objects)

### Rules

- Use `json_serializable` with `FieldRename.snake` to handle snake_case JSON automatically.
- Provide both `fromJson` factory and `toJson` method.
- **No business logic** — DTOs are pure data containers for serialization.
- DTOs and Entities **must never be the same class** — they have different responsibilities
  and different lifetimes. Merging them forces JSON concerns into the domain, which is
  a Global Rule 3 violation waiting to happen.

### Full Example

```dart
import 'package:json_annotation/json_annotation.dart';

part 'user_profile_dto.g.dart';

@JsonSerializable(fieldRename: FieldRename.snake)
class UserProfileDto {
  final String id;
  final String name;
  final String email;
  final String? avatarUrl; // maps to "avatar_url" in JSON automatically

  const UserProfileDto({
    required this.id,
    required this.name,
    required this.email,
    this.avatarUrl,
  });

  factory UserProfileDto.fromJson(Map<String, dynamic> json) =>
      _$UserProfileDtoFromJson(json);

  Map<String, dynamic> toJson() => _$UserProfileDtoToJson(this);
}
```

### Key Points

| Rule | Detail |
|---|---|
| `FieldRename.snake` | Eliminates manual `@JsonKey(name: '...')` on every field |
| No `Equatable` needed | DTOs are not compared for equality in business logic |
| No validation | Validation is the entity's job |
| Nullable fields | Use `?` for optional JSON fields so `fromJson` doesn't throw on missing keys |
| Separate class | Entity and DTO are always two separate files, two separate classes |

### pubspec Dependencies

```yaml
dependencies:
  json_annotation: ^4.9.0

dev_dependencies:
  build_runner: ^2.4.0
  json_serializable: ^6.8.0
```

Run code generation:

```bash
dart run build_runner build --delete-conflicting-outputs
```

---

## Entities vs DTOs — at a Glance

| Concern | Entity | DTO |
|---|---|---|
| Location | `domain/[feature]/entities/` | `data/[feature]/models/` |
| Extends | `Equatable` | Nothing (or plain class) |
| JSON annotations | Never | Always (`@JsonSerializable`) |
| Business logic | Allowed | Never |
| Exposed to BLoC | Yes | Never (Global Rule 3) |
| `copyWith` | Required | Not needed |
