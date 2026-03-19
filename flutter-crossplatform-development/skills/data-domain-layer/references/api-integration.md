---
description: Dio setup, Retrofit clients, interceptors, error mapping, and offline-first patterns.
---

# API Integration: Dio, Retrofit, Interceptors, and Offline-First

---

## 1. Dio Setup with BaseOptions

Create a single `Dio` instance per environment and configure it at composition root.
Never create `Dio` instances inside individual data source classes.

```dart
import 'package:dio/dio.dart';

Dio createDio({required String baseUrl}) {
  return Dio(
    BaseOptions(
      baseUrl: baseUrl,
      connectTimeout: const Duration(seconds: 10),
      receiveTimeout: const Duration(seconds: 30),
      sendTimeout: const Duration(seconds: 10),
      headers: {
        'Accept':       'application/json',
        'Content-Type': 'application/json',
      },
    ),
  )
    ..interceptors.add(AuthInterceptor(tokenStorage: getIt<TokenStorage>()))
    ..interceptors.add(RetryInterceptor(dio: /* self-reference via factory */))
    ..interceptors.add(LoggingInterceptor());
}
```

---

## 2. Retrofit `@RestApi` Client with Code Generation

Retrofit generates the HTTP implementation from an annotated abstract class.
The generated factory constructor takes the shared `Dio` instance.

```dart
import 'package:dio/dio.dart';
import 'package:retrofit/retrofit.dart';

part 'profile_api_client.g.dart';

@RestApi()
abstract class ProfileApiClient {
  factory ProfileApiClient(Dio dio, {String? baseUrl}) = _ProfileApiClient;

  @GET('/profile')
  Future<UserProfileDto> getProfile();

  @GET('/profile/{id}')
  Future<UserProfileDto> getProfileById(@Path('id') String id);

  @PUT('/profile')
  Future<UserProfileDto> updateProfile(@Body() UserProfileDto dto);

  @DELETE('/profile/{id}')
  Future<void> deleteProfile(@Path('id') String id);

  @GET('/profiles')
  Future<List<UserProfileDto>> listProfiles(
    @Query('page') int page,
    @Query('size') int size,
  );
}
```

### pubspec Dependencies

```yaml
dependencies:
  dio: ^5.4.0
  retrofit: ^4.1.0

dev_dependencies:
  build_runner: ^2.4.0
  retrofit_generator: ^8.1.0
```

Run code generation:

```bash
dart run build_runner build --delete-conflicting-outputs
```

---

## 3. Interceptors

### Auth Token Interceptor

Attaches the Bearer token to every request. If a 401 is received, attempts a token
refresh and retries the original request once.

```dart
import 'package:dio/dio.dart';

class AuthInterceptor extends Interceptor {
  final TokenStorage _tokenStorage;

  AuthInterceptor({required TokenStorage tokenStorage})
      : _tokenStorage = tokenStorage;

  @override
  Future<void> onRequest(
    RequestOptions options,
    RequestInterceptorHandler handler,
  ) async {
    final token = await _tokenStorage.readAccessToken();
    if (token != null) {
      options.headers['Authorization'] = 'Bearer $token';
    }
    handler.next(options);
  }

  @override
  Future<void> onError(
    DioException err,
    ErrorInterceptorHandler handler,
  ) async {
    if (err.response?.statusCode == 401) {
      try {
        await _refreshToken();
        final retried = await _retry(err.requestOptions);
        handler.resolve(retried);
        return;
      } catch (_) {
        // Refresh failed — let the error propagate so the repository maps it.
      }
    }
    handler.next(err);
  }

  Future<void> _refreshToken() async {
    // Call token-refresh endpoint and write new tokens to storage.
  }

  Future<Response<dynamic>> _retry(RequestOptions options) {
    final dio = Dio();
    return dio.fetch(options);
  }
}
```

### Logging Interceptor

```dart
import 'package:dio/dio.dart';

class LoggingInterceptor extends Interceptor {
  @override
  void onRequest(RequestOptions options, RequestInterceptorHandler handler) {
    // In debug mode only — never log tokens in production.
    assert(() {
      // ignore: avoid_print
      print('[API] ${options.method} ${options.uri}');
      return true;
    }());
    handler.next(options);
  }

  @override
  void onResponse(Response response, ResponseInterceptorHandler handler) {
    assert(() {
      // ignore: avoid_print
      print('[API] ${response.statusCode} ${response.requestOptions.uri}');
      return true;
    }());
    handler.next(response);
  }

  @override
  void onError(DioException err, ErrorInterceptorHandler handler) {
    assert(() {
      // ignore: avoid_print
      print('[API] ERROR ${err.response?.statusCode} ${err.requestOptions.uri}');
      return true;
    }());
    handler.next(err);
  }
}
```

### Retry Interceptor

Retries idempotent requests (GET, HEAD, PUT) on connection-level failures,
with exponential back-off.

```dart
import 'package:dio/dio.dart';

class RetryInterceptor extends Interceptor {
  final Dio _dio;
  final int maxRetries;

  RetryInterceptor({required Dio dio, this.maxRetries = 3}) : _dio = dio;

  @override
  Future<void> onError(
    DioException err,
    ErrorInterceptorHandler handler,
  ) async {
    final options    = err.requestOptions;
    final attempt    = options.extra['_retryCount'] as int? ?? 0;
    final idempotent = ['GET', 'HEAD', 'PUT'].contains(options.method);
    final retriable  = err.type == DioExceptionType.connectionTimeout ||
                       err.type == DioExceptionType.receiveTimeout ||
                       err.type == DioExceptionType.connectionError;

    if (idempotent && retriable && attempt < maxRetries) {
      options.extra['_retryCount'] = attempt + 1;
      await Future<void>.delayed(Duration(seconds: 1 << attempt)); // 1s, 2s, 4s
      try {
        handler.resolve(await _dio.fetch(options));
        return;
      } on DioException catch (e) {
        handler.next(e);
        return;
      }
    }
    handler.next(err);
  }
}
```

---

## 4. Error Mapping: DioException → Domain Exceptions

All `DioException` handling must happen inside the repository implementation.
The domain layer and BLoC layer must never import `package:dio`.

```dart
// Inside ProfileRepositoryImpl
Exception _mapException(DioException e) {
  // Status-code mapping takes priority.
  final statusCode = e.response?.statusCode;
  if (statusCode != null) {
    return switch (statusCode) {
      400 => ProfileException('Bad request', code: 'BAD_REQUEST'),
      401 => AuthException('Unauthorized', code: 'UNAUTHORIZED'),
      403 => AuthException('Forbidden', code: 'FORBIDDEN'),
      404 => ProfileException('Not found', code: 'NOT_FOUND'),
      409 => ProfileException('Conflict', code: 'CONFLICT'),
      422 => ProfileException('Validation error', code: 'UNPROCESSABLE'),
      >= 500 && < 600 => ProfileException('Server error', code: 'SERVER_ERROR'),
      _ => ProfileException('HTTP error $statusCode'),
    };
  }

  // Network-level errors.
  return switch (e.type) {
    DioExceptionType.connectionTimeout =>
      ProfileException('Connection timeout', code: 'TIMEOUT'),
    DioExceptionType.receiveTimeout =>
      ProfileException('Receive timeout', code: 'TIMEOUT'),
    DioExceptionType.sendTimeout =>
      ProfileException('Send timeout', code: 'TIMEOUT'),
    DioExceptionType.cancel =>
      ProfileException('Request cancelled', code: 'CANCELLED'),
    DioExceptionType.connectionError =>
      ProfileException('No internet connection', code: 'NO_CONNECTION'),
    _ => ProfileException('Network error: ${e.message}'),
  };
}
```

---

## 5. CancelToken for Request Cancellation

Pass a `CancelToken` through to the API client when you need to cancel in-flight
requests — for example when the user navigates away mid-search.

```dart
// DAO / repository method accepts an optional token.
Future<List<UserProfile>> searchProfiles(
  String query, {
  CancelToken? cancelToken,
}) async {
  try {
    final dtos = await _apiClient.searchProfiles(
      query,
      cancelToken: cancelToken,
    );
    return dtos.map(_mapper.toEntity).toList();
  } on DioException catch (e) {
    if (e.type == DioExceptionType.cancel) {
      return [];          // Treat cancellation as an empty result.
    }
    throw _mapException(e);
  }
}
```

```dart
// BLoC event handler — cancel the previous token on new input.
CancelToken? _searchToken;

Future<void> _onSearchRequested(
  SearchRequested event,
  Emitter<SearchState> emit,
) async {
  _searchToken?.cancel();
  _searchToken = CancelToken();

  emit(const SearchState.loading());
  try {
    final results = await _profileRepository.searchProfiles(
      event.query,
      cancelToken: _searchToken,
    );
    emit(SearchState.success(results));
  } on DomainException catch (e) {
    emit(SearchState.failure(e.message));
  }
}
```

---

## 6. Offline-First Patterns

### Cache-Then-Network

Serve cached data immediately while a fresh fetch runs in the background.
The repository emits the cached entity, then silently updates the cache.

```dart
@override
Future<UserProfile> getProfile() async {
  try {
    final cached = await _localStorage.getCachedProfile();
    if (cached != null) {
      unawaited(_refreshProfile()); // Background refresh; caller gets cached data now.
      return _mapper.toEntity(cached);
    }
    return await _fetchAndCache();
  } on DioException catch (e) {
    throw _mapException(e);
  }
}
```

### Queue Failed Requests for Later Sync

Store failed mutations in a Drift table. A background sync service replays the queue
when connectivity is restored.

```dart
// Table: pending_operations
class PendingOperations extends Table {
  IntColumn  get id        => integer().autoIncrement()();
  TextColumn get type      => text()();        // e.g. 'update_profile'
  TextColumn get payload   => text()();        // JSON-encoded DTO
  IntColumn  get attempts  => integer().withDefault(const Constant(0))();
  DateTimeColumn get createdAt => dateTime()();
}

// Repository: queue on failure
@override
Future<void> updateProfile(UserProfile profile) async {
  try {
    final dto = _mapper.toDto(profile);
    await _apiClient.updateProfile(dto);
    await _localStorage.cacheProfile(dto);
  } on DioException catch (e) {
    if (e.type == DioExceptionType.connectionError) {
      // Save for later rather than failing immediately.
      await _syncQueue.enqueue(
        type: 'update_profile',
        payload: jsonEncode(_mapper.toDto(profile).toJson()),
      );
      return;
    }
    throw _mapException(e);
  }
}
```

### Offline-First Checklist

✅ Cache reads to Drift or SharedPreferences on successful network responses
✅ Return cached data when the network is unavailable
✅ Queue write operations that fail due to connectivity errors
✅ Expose connectivity-specific domain exceptions (`code: 'NO_CONNECTION'`)
❌ Never throw blindly on network errors — always check if cached data can satisfy the request first

---

## Anti-Patterns

1. **Catching `DioException` in a BLoC** — the repository is the only permitted catch site.
   If a `DioException` reaches the BLoC, the repository boundary is broken.

2. **Creating `Dio` inside a data source class** — `Dio` must be a singleton created at
   the composition root and injected.

3. **Calling API endpoints directly from the BLoC** — the BLoC must only call repository
   interface methods.

4. **Logging full request/response bodies in production** — use `assert()` blocks in the
   logging interceptor so they compile out in release mode.

5. **Not cancelling in-flight requests on BLoC close** — override `close()` on the BLoC
   and call `cancelToken.cancel()` to avoid dangling futures.

6. **Hardcoding base URLs in Retrofit `@RestApi(baseUrl: '...')`** — pass the URL via the
   `Dio.BaseOptions` so it can be swapped per environment without regenerating code.
