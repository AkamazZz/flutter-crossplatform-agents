---
description: >
  Secure storage with flutter_secure_storage, keychain/keystore integration, and token management.
---

# Secure Storage

Sensitive data must never rest in plaintext. Flutter provides `flutter_secure_storage` as a cross-platform API over iOS Keychain and Android KeyStore, giving hardware-backed encryption for credentials and tokens.

## Setup

```yaml
# pubspec.yaml
dependencies:
  flutter_secure_storage: ^9.2.2
```

**Android minimum SDK** — EncryptedSharedPreferences requires API 23+:
```groovy
// android/app/build.gradle
android {
    defaultConfig {
        minSdkVersion 23
    }
}
```

**iOS** — no additional configuration needed; uses Keychain by default.

## Reading, Writing, and Deleting

```dart
import 'package:flutter_secure_storage/flutter_secure_storage.dart';

class SecureStorageService {
  static const _storage = FlutterSecureStorage(
    // Android: use EncryptedSharedPreferences backed by KeyStore
    aOptions: AndroidOptions(encryptedSharedPreferences: true),
    // iOS: accessible after first unlock, survives app reinstall
    iOptions: IOSOptions(
      accessibility: KeychainAccessibility.first_unlock,
      synchronizable: false,  // do NOT sync to iCloud
    ),
  );

  // Write
  static Future<void> write(String key, String value) =>
      _storage.write(key: key, value: value);

  // Read (returns null if not found)
  static Future<String?> read(String key) =>
      _storage.read(key: key);

  // Check existence without reading value
  static Future<bool> containsKey(String key) =>
      _storage.containsKey(key: key);

  // Delete single entry
  static Future<void> delete(String key) =>
      _storage.delete(key: key);

  // Delete all entries (use on logout)
  static Future<void> deleteAll() =>
      _storage.deleteAll();

  // Read all entries (use carefully — avoid in hot paths)
  static Future<Map<String, String>> readAll() =>
      _storage.readAll();
}
```

## iOS Keychain

The Keychain stores data in encrypted hardware-backed storage. Key accessibility options:

| `KeychainAccessibility` | Accessible when |
|---|---|
| `unlocked` | Device is unlocked (default) |
| `first_unlock` | After first unlock since boot; survives background |
| `always` | Always, even when locked (avoid for sensitive data) |
| `unlocked_this_device` | Unlocked + not transferred to new device |
| `first_unlock_this_device` | After first unlock + not transferred |

Recommendation: use `first_unlock` for auth tokens (allows background refresh) and `unlocked` for highly sensitive keys.

**Do not set `synchronizable: true`** unless you explicitly want iCloud Keychain sync, which moves data off-device.

## Android KeyStore and EncryptedSharedPreferences

With `AndroidOptions(encryptedSharedPreferences: true)`:
- Data is encrypted with AES-256-GCM
- The encryption key is stored in the Android KeyStore, which is hardware-backed on most modern devices
- The encrypted file lives in the app's private data directory

Legacy `AndroidOptions` (without EncryptedSharedPreferences) uses a custom RSA+AES scheme stored in SharedPreferences — use the EncryptedSharedPreferences option for all new projects.

**Keystore key invalidation:** if the user removes their screen lock PIN/biometric, Android may invalidate KeyStore keys. Handle this gracefully:
```dart
try {
  final token = await SecureStorageService.read('access_token');
} on PlatformException catch (e) {
  if (e.code == 'Exception') {
    // Key invalidated — force re-login
    await SecureStorageService.deleteAll();
    navigateToLogin();
  }
}
```

## Token Lifecycle: Access/Refresh, Rotation, Expiry

```dart
class TokenRepository {
  static const _accessTokenKey  = 'access_token';
  static const _refreshTokenKey = 'refresh_token';
  static const _expiryKey       = 'token_expiry_utc';

  // Store tokens after successful login
  Future<void> saveTokens({
    required String accessToken,
    required String refreshToken,
    required DateTime expiry,
  }) async {
    await Future.wait([
      SecureStorageService.write(_accessTokenKey,  accessToken),
      SecureStorageService.write(_refreshTokenKey, refreshToken),
      SecureStorageService.write(_expiryKey, expiry.toUtc().toIso8601String()),
    ]);
  }

  // Get a valid access token, refreshing if near expiry
  Future<String?> getValidAccessToken() async {
    final expiryStr = await SecureStorageService.read(_expiryKey);
    if (expiryStr == null) return null;

    final expiry = DateTime.parse(expiryStr).toUtc();
    final now    = DateTime.now().toUtc();

    // Refresh 60 seconds before actual expiry to avoid race conditions
    if (now.isAfter(expiry.subtract(const Duration(seconds: 60)))) {
      return _refreshAccessToken();
    }
    return SecureStorageService.read(_accessTokenKey);
  }

  Future<String?> _refreshAccessToken() async {
    final refreshToken = await SecureStorageService.read(_refreshTokenKey);
    if (refreshToken == null) return null;

    try {
      final response = await authApi.refresh(refreshToken: refreshToken);
      await saveTokens(
        accessToken:  response.accessToken,
        refreshToken: response.refreshToken,  // rotate refresh token
        expiry:       response.expiry,
      );
      return response.accessToken;
    } on UnauthorizedException {
      await clearTokens();   // refresh token expired — force re-login
      return null;
    }
  }

  Future<void> clearTokens() => SecureStorageService.deleteAll();
}
```

**Token lifetime recommendations:**
- Access token: 15 minutes
- Refresh token: 7–30 days (rotate on each use)
- Implement refresh token rotation: each use of a refresh token issues a new one and invalidates the old one

## Encryption at Rest for Local Databases

If you store structured data locally (e.g., with `sqflite`), encrypt the database file:

```dart
// sqflite_sqlcipher or drift with SQLCipher
import 'package:sqflite_sqlcipher/sqflite.dart';

final dbKey = await SecureStorageService.read('db_key')
    ?? await _generateAndStoreDbKey();

final db = await openDatabase(
  path,
  password: dbKey,
  version: 1,
  onCreate: (db, version) { /* create tables */ },
);

Future<String> _generateAndStoreDbKey() async {
  final key = base64Url.encode(List<int>.generate(32, (_) => Random.secure().nextInt(256)));
  await SecureStorageService.write('db_key', key);
  return key;
}
```

## Secure Deletion

When a user logs out or requests account deletion, delete all stored data:

```dart
Future<void> onLogout() async {
  await SecureStorageService.deleteAll();      // tokens, keys
  await preferences.clear();                   // non-sensitive prefs
  await localDatabase.deleteEverything();       // cached data
  await cacheManager.emptyCache();              // image/file cache
}
```

On iOS, `deleteAll()` removes Keychain entries with the app's service name. On Android, it clears the EncryptedSharedPreferences file. Neither platform guarantees overwriting the underlying flash pages — that is handled by the OS-level secure enclave and file system encryption.

## Decision Guide: SecureStorage vs SharedPreferences

| Data type | Storage choice | Reason |
|---|---|---|
| Access token | `flutter_secure_storage` | Credential — must be encrypted |
| Refresh token | `flutter_secure_storage` | Long-lived credential |
| Encryption/signing keys | `flutter_secure_storage` | Key material |
| User passwords (if cached) | `flutter_secure_storage` | Never store; use tokens instead |
| Biometric-protected secrets | `flutter_secure_storage` | Hardware-backed |
| User ID (non-secret) | `shared_preferences` | Not sensitive, fine unencrypted |
| UI preferences (theme, locale) | `shared_preferences` | Not sensitive |
| Feature flags | `shared_preferences` | Not sensitive |
| Analytics opt-out | `shared_preferences` | Not sensitive |
| Last sync timestamp | `shared_preferences` | Not sensitive |

**Rule:** if the data could be used to authenticate, decrypt, or identify the user, it belongs in `flutter_secure_storage`. Everything else can go in `shared_preferences`.
