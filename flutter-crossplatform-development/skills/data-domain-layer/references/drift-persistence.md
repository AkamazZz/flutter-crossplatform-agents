---
description: Local persistence with Drift, SharedPreferences, and flutter_secure_storage.
---

# Drift Persistence and Local Storage

> Global Rule 6: **No Hive.** Use Drift for structured data, SharedPreferences for
> simple key-value settings, and flutter_secure_storage for tokens and credentials.

---

## pubspec Setup

```yaml
dependencies:
  drift: ^2.18.0
  sqlite3_flutter_libs: ^0.5.0   # bundles SQLite for Android/iOS
  path_provider: ^2.1.0
  path: ^1.9.0
  shared_preferences: ^2.2.0
  flutter_secure_storage: ^9.0.0

dev_dependencies:
  drift_dev: ^2.18.0
  build_runner: ^2.4.0
```

Run code generation after any schema change:

```bash
dart run build_runner build --delete-conflicting-outputs
```

---

## 1. Database Class and `@DriftDatabase` Annotation

The database class is the single entry point for all Drift operations.
Declare every table and DAO it owns in the annotation.

```dart
import 'dart:io';
import 'package:drift/drift.dart';
import 'package:drift/native.dart';
import 'package:path/path.dart' as p;
import 'package:path_provider/path_provider.dart';

part 'app_database.g.dart';

@DriftDatabase(tables: [UserProfiles, CachedOrders], daos: [ProfileDao, OrderDao])
class AppDatabase extends _$AppDatabase {
  AppDatabase() : super(_openConnection());

  @override
  int get schemaVersion => 2;

  @override
  MigrationStrategy get migration => MigrationStrategy(
    stepByStep: stepByStep,
    onCreate: (m) => m.createAll(),
  );
}

LazyDatabase _openConnection() {
  return LazyDatabase(() async {
    final dir  = await getApplicationDocumentsDirectory();
    final file = File(p.join(dir.path, 'app.db'));
    return NativeDatabase.createInBackground(file);
  });
}
```

---

## 2. Table Definitions

Each table is a plain Dart class that extends `Table`.
Column types are declared with typed builders; names map to snake_case SQL columns automatically.

```dart
class UserProfiles extends Table {
  TextColumn get id       => text()();
  TextColumn get name     => text()();
  TextColumn get email    => text()();
  TextColumn get avatarUrl => text().nullable()();
  DateTimeColumn get updatedAt => dateTime()();

  @override
  Set<Column> get primaryKey => {id};
}

class CachedOrders extends Table {
  TextColumn get id        => text()();
  TextColumn get userId    => text()();
  RealColumn get total     => real()();
  TextColumn get status    => text()();
  BoolColumn get synced    => boolean().withDefault(const Constant(false))();
  DateTimeColumn get createdAt => dateTime()();

  @override
  Set<Column> get primaryKey => {id};
}
```

### Column Builders Quick Reference

- `String` → `text()` — e.g. `text().nullable()()`
- `int` → `integer()` — e.g. `integer().withDefault(const Constant(0))()`
- `double` → `real()` — e.g. `real()()`
- `bool` → `boolean()` — e.g. `boolean().withDefault(const Constant(false))()`
- `DateTime` → `dateTime()` — e.g. `dateTime()()`
- `Uint8List` → `blob()` — e.g. `blob()()`

---

## 3. DAOs with `@DriftAccessor`

DAOs keep query logic out of the database class. Each DAO owns one domain area.

```dart
part 'profile_dao.g.dart';

@DriftAccessor(tables: [UserProfiles])
class ProfileDao extends DatabaseAccessor<AppDatabase> with _$ProfileDaoMixin {
  ProfileDao(super.db);

  // ── Single row ────────────────────────────────────────────────
  Future<UserProfile?> getById(String id) =>
      (select(userProfiles)..where((t) => t.id.equals(id)))
          .getSingleOrNull();

  // ── All rows ──────────────────────────────────────────────────
  Future<List<UserProfile>> getAll() => select(userProfiles).get();

  // ── Insert or replace ─────────────────────────────────────────
  Future<void> upsert(UserProfilesCompanion entry) =>
      into(userProfiles).insertOnConflictUpdate(entry);

  // ── Delete ────────────────────────────────────────────────────
  Future<int> deleteById(String id) =>
      (delete(userProfiles)..where((t) => t.id.equals(id))).go();
}
```

---

## 4. Schema Migrations

Use `stepByStep` migrations for safe, incremental schema upgrades.
Each step runs exactly the changes introduced by that schema version.

```dart
// In AppDatabase.migration (see above)
MigrationStrategy get migration => MigrationStrategy(
  stepByStep: stepByStep,
  onCreate: (m) => m.createAll(),
);

// Declare per-version migration steps in a separate file:
// lib/data/database/migrations.dart
final stepByStep = <int, Future<void> Function(Migrator, Schema)>{
  // Version 2: add avatarUrl column to user_profiles
  2: (m, schema) async {
    await m.addColumn(
      schema.userProfiles,
      schema.userProfiles.avatarUrl,
    );
  },
};
```

### Migration Rules

- Never modify an existing migration step — add a new version instead.
- Increment `schemaVersion` in the database class for every structural change.
- Test migrations with `drift_dev`'s schema test utilities before shipping.

---

## 5. Reactive Queries with `watch()` → Stream

Use `watch()` instead of `get()` to receive a new emission whenever the underlying
table data changes. Wire this to a repository `Stream` that the BLoC observes.

```dart
// DAO
Stream<List<UserProfile>> watchAll() => select(userProfiles).watch();

Stream<UserProfile?> watchById(String id) =>
    (select(userProfiles)..where((t) => t.id.equals(id)))
        .watchSingleOrNull();

// Repository implementation
@override
Stream<UserProfile> watchProfile(String id) {
  return _profileDao
      .watchById(id)
      .where((row) => row != null)
      .map((row) => _mapper.toEntity(row!));
}
```

Drift's `watch()` streams are backed by SQLite triggers and emit only when data actually
changes — no polling required.

---

## 6. Transactions

Wrap multiple writes in a transaction to guarantee atomicity.

```dart
Future<void> saveOrderWithItems(OrderDto order, List<OrderItemDto> items) {
  return db.transaction(() async {
    await db.into(db.cachedOrders).insertOnConflictUpdate(
      CachedOrdersCompanion.insert(
        id: order.id,
        userId: order.userId,
        total: order.total,
        status: order.status,
        createdAt: DateTime.now(),
      ),
    );

    for (final item in items) {
      await db.into(db.orderItems).insertOnConflictUpdate(
        OrderItemsCompanion.insert(
          id: item.id,
          orderId: order.id,
          quantity: item.quantity,
          price: item.price,
        ),
      );
    }
  });
}
```

If any statement inside `transaction()` throws, all writes are rolled back.

---

## 7. SharedPreferences for Simple Key-Value Data

Use SharedPreferences for booleans, strings, and integers that do not require relational
queries or reactivity.

```dart
class AppSettingsStorage {
  static const _keyOnboardingComplete = 'onboarding_complete';
  static const _keySelectedLocale     = 'selected_locale';

  final SharedPreferences _prefs;
  const AppSettingsStorage(this._prefs);

  bool get isOnboardingComplete =>
      _prefs.getBool(_keyOnboardingComplete) ?? false;

  Future<void> setOnboardingComplete() =>
      _prefs.setBool(_keyOnboardingComplete, true);

  String? get selectedLocale => _prefs.getString(_keySelectedLocale);

  Future<void> setSelectedLocale(String locale) =>
      _prefs.setString(_keySelectedLocale, locale);
}
```

SharedPreferences reads are synchronous after the initial async `getInstance()` call.
Store the instance at app startup and inject it into storage classes.

---

## 8. flutter_secure_storage for Tokens and Credentials

Use flutter_secure_storage for any data that must not appear in device backups or
be readable by other apps: access tokens, refresh tokens, PIN codes, private keys.

```dart
import 'package:flutter_secure_storage/flutter_secure_storage.dart';

class TokenStorage {
  static const _keyAccessToken  = 'access_token';
  static const _keyRefreshToken = 'refresh_token';

  final FlutterSecureStorage _storage;
  const TokenStorage(this._storage);

  Future<String?> readAccessToken() =>
      _storage.read(key: _keyAccessToken);

  Future<void> writeAccessToken(String token) =>
      _storage.write(key: _keyAccessToken, value: token);

  Future<String?> readRefreshToken() =>
      _storage.read(key: _keyRefreshToken);

  Future<void> writeRefreshToken(String token) =>
      _storage.write(key: _keyRefreshToken, value: token);

  Future<void> clearAll() => _storage.deleteAll();
}
```

flutter_secure_storage uses the Android Keystore and iOS Keychain under the hood.
Never store tokens in SharedPreferences — they are written to disk in plain text.

---

## 9. Decision Guide: Drift vs SharedPreferences vs SecureStorage

- Structured relational data (users, orders, products) → **Drift**
- Reactive data that the UI streams → **Drift** (`watch()`)
- Offline queue of pending requests → **Drift**
- Simple flags and user preferences → **SharedPreferences**
- Last-selected tab, theme choice, locale → **SharedPreferences**
- Access tokens, refresh tokens → **flutter_secure_storage**
- PINs, passwords, private keys → **flutter_secure_storage**
- Data that must survive process restart but not be sensitive → **SharedPreferences** or **Drift**
- Any sensitive credential → **flutter_secure_storage** only

---

## No Hive (Global Rule 6)

Hive is **not permitted** on this project. Do not add `hive`, `hive_flutter`,
or `hive_generator` as dependencies. If you encounter existing Hive code during
migration, replace it with Drift (structured) or SharedPreferences (simple key-value)
and update the DI wiring accordingly.
