---
description: Mapper classes, repository implementation, error handling, and DI for the Data layer.
---

# Repository Pattern: Mappers, Implementation, Error Handling, and DI

> Cross-reference: for DI container setup and `CompositionRoot` see
> `architecture-patterns/dependency-injection`.

---

## 6. Data Layer: Mapper Classes

Mapper classes perform the translation between DTOs (infrastructure) and Entities (domain).
They are the sole place where this translation should happen.

### When to Use a Mapper Class

- Bidirectional mapping is needed (DTO ↔ Entity).
- Transformation logic is non-trivial (date parsing, status enums, nested objects).
- Mapping spans multiple DTOs or Entity types.
- You want the mapping logic to be independently testable and reusable.

### Simple Bidirectional Mapper

```dart
class UserProfileMapper {
  const UserProfileMapper();

  UserProfile toEntity(UserProfileDto dto) {
    return UserProfile(id: dto.id, name: dto.name, email: dto.email);
  }

  UserProfileDto toDto(UserProfile entity) {
    return UserProfileDto(
      id: entity.id,
      name: entity.name,
      email: entity.email,
    );
  }

  List<UserProfile> toEntityList(List<UserProfileDto> dtos) =>
      dtos.map(toEntity).toList();
}
```

### Complex Mapper with Injected Dependencies

When an entity contains nested objects that each have their own mapper, inject those
sub-mappers via the constructor rather than instantiating them internally.

```dart
class OrderMapper {
  final ProductMapper _productMapper;

  const OrderMapper({required ProductMapper productMapper})
      : _productMapper = productMapper;

  Order toEntity(OrderDto dto) {
    return Order(
      id: dto.id,
      items: dto.items.map(_productMapper.toEntity).toList(),
      total: _calculateTotal(dto.items),
      status: _mapStatus(dto.statusCode),
    );
  }

  double _calculateTotal(List<ProductDto> items) =>
      items.fold(0.0, (sum, item) => sum + item.price * item.quantity);

  OrderStatus _mapStatus(String code) => switch (code) {
    'pending'    => OrderStatus.pending,
    'processing' => OrderStatus.processing,
    'completed'  => OrderStatus.completed,
    _            => OrderStatus.unknown,
  };
}
```

### Mapper Rules

| Rule | Detail |
|---|---|
| One mapper per entity type | `UserProfileMapper`, `OrderMapper`, etc. |
| `const` constructor | Mark mappers `const` when they have no mutable state |
| No business logic | Mappers translate structure only — validation stays in the entity |
| No network/DB calls | Mappers are pure functions; they never perform I/O |

---

## 7. Data Layer: Data Sources

Data sources are single-responsibility classes that handle exactly one data origin:
an API, a local database, or a cache. They work exclusively with DTOs.

### Rules

- Handle one specific data source (API, DB, or cache) — never combine two in one class.
- Receive clients/storage via constructor injection.
- **No domain entity awareness** — accept and return DTOs only.
- **No business logic**.

### Retrofit API Client

```dart
import 'package:dio/dio.dart';
import 'package:retrofit/retrofit.dart';

part 'profile_api_client.g.dart';

@RestApi(baseUrl: '/api')
abstract class ProfileApiClient {
  factory ProfileApiClient(Dio dio, {String? baseUrl}) = _ProfileApiClient;

  @GET('/profile')
  Future<UserProfileDto> getProfile();

  @PUT('/profile')
  Future<void> updateProfile(@Body() UserProfileDto dto);

  @DELETE('/profile/{id}')
  Future<void> deleteProfile(@Path('id') String id);
}
```

### Local Storage (SharedPreferences)

```dart
import 'package:shared_preferences/shared_preferences.dart';
import 'dart:convert';

class ProfileLocalStorage {
  final SharedPreferences _prefs;
  const ProfileLocalStorage(this._prefs);

  Future<UserProfileDto?> getCachedProfile() async {
    final json = _prefs.getString('cached_profile');
    if (json == null) return null;
    return UserProfileDto.fromJson(jsonDecode(json) as Map<String, dynamic>);
  }

  Future<void> cacheProfile(UserProfileDto dto) async {
    await _prefs.setString('cached_profile', jsonEncode(dto.toJson()));
  }

  Future<void> clearCache() async {
    await _prefs.remove('cached_profile');
  }
}
```

---

## 8. Data Layer: Repository Implementation

The repository implementation ties together API clients, local storage, and mappers.
It is the **only** place where DTOs are converted to entities, and the **only** place where
infrastructure exceptions are caught and remapped to domain exceptions.

### Rules

- Mark the class `@immutable` — all fields are `final`.
- Inject all data sources and mappers via the constructor.
- Map DTOs to Entities before returning — never return a DTO from a public method.
- Own the caching strategy (cache-then-network, network-then-cache, etc.).
- Catch `DioException` here; rethrow as a `DomainException` subclass.

### Full Example

```dart
import 'package:meta/meta.dart';

@immutable
class ProfileRepositoryImpl implements IProfileRepository {
  final ProfileApiClient _apiClient;
  final ProfileLocalStorage _localStorage;
  final UserProfileMapper _mapper;

  const ProfileRepositoryImpl({
    required ProfileApiClient apiClient,
    required ProfileLocalStorage localStorage,
    UserProfileMapper mapper = const UserProfileMapper(),
  })  : _apiClient = apiClient,
        _localStorage = localStorage,
        _mapper = mapper;

  @override
  Future<UserProfile> getProfile() async {
    try {
      // Cache-first: serve stale data immediately, refresh in background.
      final cached = await _localStorage.getCachedProfile();
      if (cached != null) {
        _refreshProfile(); // fire-and-forget
        return _mapper.toEntity(cached);
      }

      final dto = await _apiClient.getProfile();
      await _localStorage.cacheProfile(dto);
      return _mapper.toEntity(dto);
    } on DioException catch (e) {
      throw _mapDioException(e);
    } on Exception catch (e) {
      throw ProfileException('Failed to fetch profile: $e');
    }
  }

  @override
  Future<void> updateProfile(UserProfile profile) async {
    try {
      final dto = _mapper.toDto(profile);
      await _apiClient.updateProfile(dto);
      await _localStorage.cacheProfile(dto);
    } on DioException catch (e) {
      throw _mapDioException(e);
    }
  }

  Future<void> _refreshProfile() async {
    try {
      final dto = await _apiClient.getProfile();
      await _localStorage.cacheProfile(dto);
    } catch (_) {
      // Silent fail — background refresh must not surface errors to the caller.
    }
  }

  Exception _mapDioException(DioException e) => switch (e.response?.statusCode) {
    401 => AuthException('Unauthorized', code: 'UNAUTHORIZED'),
    404 => ProfileException('Not found', code: 'NOT_FOUND'),
    500 => ProfileException('Server error', code: 'SERVER_ERROR'),
    _   => ProfileException('Network error: ${e.message}'),
  };
}
```

---

## 9. Error Handling

### Domain Exception Hierarchy

Define a base `DomainException` in the domain layer, then create feature-specific subtypes
in each feature package. The BLoC layer catches `DomainException` subtypes — it never
sees `DioException`.

```dart
// domain/core/lib/exceptions/domain_exception.dart
abstract class DomainException implements Exception {
  final String message;
  final String? code;

  const DomainException(this.message, {this.code});

  @override
  String toString() =>
      'DomainException: $message${code != null ? ' ($code)' : ''}';
}

// domain/profile/lib/exceptions/profile_exception.dart
class ProfileException extends DomainException {
  const ProfileException(super.message, {super.code});
}

// domain/auth/lib/exceptions/auth_exception.dart
class AuthException extends DomainException {
  const AuthException(super.message, {super.code});
}
```

### Exception Mapping from DioException

Map by HTTP status code first, then fall back to `DioExceptionType` for network-level errors.

```dart
Exception _mapException(dynamic error) {
  if (error is DioException) {
    // HTTP status code takes priority.
    return switch (error.response?.statusCode) {
      401 => AuthException('Unauthorized', code: 'UNAUTHORIZED'),
      404 => ProfileException('Not found', code: 'NOT_FOUND'),
      500 => ProfileException('Server error', code: 'SERVER_ERROR'),
      _   => switch (error.type) {
        DioExceptionType.connectionTimeout =>
          ProfileException('Connection timeout', code: 'TIMEOUT'),
        DioExceptionType.cancel =>
          ProfileException('Request cancelled', code: 'CANCELLED'),
        _ => ProfileException('Network error: ${error.message}'),
      },
    };
  }
  return ProfileException('Unknown error: $error');
}
```

### Rules

| Rule | Detail |
|---|---|
| Catch at boundary | Only the repository implementation catches infrastructure exceptions |
| Rethrow as domain | Always rethrow as a `DomainException` subclass — never rethrow raw |
| BLoC handles domain | BLoC `on<>` handlers catch `DomainException`; they never import `DioException` |
| `DioException` in BLoC | This is an anti-pattern — fix it by mapping in the repository |

---

## 10. Dependency Injection Setup

Wiring follows the Dependency Inversion Principle: BLoC depends on the interface;
the concrete implementation is registered in the composition root.

```dart
void setupDependencies() {
  // ── Networking ────────────────────────────────────────────────
  final dio = Dio(
    BaseOptions(
      baseUrl: 'https://api.example.com',
      connectTimeout: const Duration(seconds: 10),
      receiveTimeout: const Duration(seconds: 10),
    ),
  );
  // Add interceptors here (auth, logging, retry) — see api-integration.md.

  // ── Data sources ──────────────────────────────────────────────
  final apiClient    = ProfileApiClient(dio);
  final localStorage = ProfileLocalStorage(await SharedPreferences.getInstance());

  // ── Mappers ───────────────────────────────────────────────────
  final mapper = const UserProfileMapper();

  // ── Repository ────────────────────────────────────────────────
  // Register against the interface so the BLoC receives IProfileRepository,
  // not the concrete type.
  final IProfileRepository profileRepository = ProfileRepositoryImpl(
    apiClient: apiClient,
    localStorage: localStorage,
    mapper: mapper,
  );
}
```

For a full `CompositionRoot` and scoped DI patterns see
`architecture-patterns/dependency-injection`.

---

## 11. Best Practices Checklist

| | Rule |
|---|---|
| ✅ | Keep the domain layer pure — no external package imports other than `equatable` and `meta` |
| ✅ | Use interfaces for repositories (Dependency Inversion) |
| ✅ | Map DTOs to entities at the repository boundary — never before, never after |
| ✅ | Handle errors at the data layer; expose only `DomainException` subtypes |
| ✅ | Implement caching strategy inside the repository implementation |
| ✅ | Mark repository implementations and mappers `@immutable` |
| ✅ | Use mapper classes for complex or bidirectional conversions |
| ✅ | Prefer composition over inheritance for mappers |
| ❌ | Never expose DTOs to the BLoC layer (Global Rule 3) |
| ❌ | Never import domain entities inside data source classes |
| ❌ | Never put business logic in repositories, DTOs, or mappers |
| ❌ | Never catch `DioException` in a BLoC event handler |
