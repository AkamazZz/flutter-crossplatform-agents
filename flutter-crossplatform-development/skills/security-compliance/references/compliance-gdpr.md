---
description: >
  GDPR compliance, privacy-first development, data minimization, and consent management.
---

# Compliance & GDPR

The General Data Protection Regulation (GDPR) and similar frameworks (CCPA, LGPD, PIPL) require apps to handle personal data with purpose, transparency, and user control. Privacy-by-design means building these constraints into the architecture rather than retrofitting them.

## GDPR Core Principles

- Lawful basis — process data only with consent, contract necessity, legal obligation, vital interest, public task, or legitimate interest
- Purpose limitation — collect data for specified, explicit purposes; do not reuse for incompatible purposes
- Data minimization — collect only what is strictly necessary for the stated purpose
- Accuracy — keep data up to date; allow users to correct it
- Storage limitation — delete data when it is no longer needed for its purpose
- Integrity & confidentiality — protect data with appropriate technical measures (encryption, access controls)
- Accountability — demonstrate compliance; maintain records of processing activities

## Privacy-by-Design

Apply these constraints from the start of feature development:

1. **Default to privacy** — if a feature can work without collecting a data point, do not collect it
2. **No PII in analytics** — use pseudonymous user IDs (UUIDs), never email addresses, names, or phone numbers, in analytics events
3. **No PII in logs** — log event names and codes, never user-identifiable content
4. **Minimize data in transit** — request only the fields you display; do not fetch full user profiles when only a name is needed
5. **Data isolation** — store PII in a dedicated service/table with strict access control, not scattered across tables
6. **Retention policies** — define and enforce maximum retention periods for each data category at the database level

```dart
// Anti-pattern: logging PII
debugPrint('User logged in: ${user.email}');  // NEVER do this

// Correct: log pseudonymous events only
debugPrint('User logged in: userId=${user.id.substring(0, 8)}...');
// Or use structured logging with a redaction layer
logger.info('auth.login_success', properties: {'user_id': user.anonymousId});
```

## Consent Management

Collect consent before processing any non-essential data (analytics, marketing, personalization).

```dart
class ConsentRepository {
  static const _analyticsConsentKey   = 'consent_analytics';
  static const _marketingConsentKey   = 'consent_marketing';
  static const _consentVersionKey     = 'consent_version';
  static const _currentConsentVersion = '2024-01';

  final SharedPreferences _prefs;

  ConsentRepository(this._prefs);

  bool get analyticsConsented =>
      _prefs.getBool(_analyticsConsentKey) ?? false;

  bool get marketingConsented =>
      _prefs.getBool(_marketingConsentKey) ?? false;

  bool get consentIsUpToDate =>
      _prefs.getString(_consentVersionKey) == _currentConsentVersion;

  Future<void> saveConsent({
    required bool analytics,
    required bool marketing,
  }) async {
    await Future.wait([
      _prefs.setBool(_analyticsConsentKey,  analytics),
      _prefs.setBool(_marketingConsentKey,  marketing),
      _prefs.setString(_consentVersionKey,  _currentConsentVersion),
    ]);
    _applyConsent(analytics: analytics, marketing: marketing);
  }

  void _applyConsent({required bool analytics, required bool marketing}) {
    if (analytics) {
      AnalyticsService.enable();
    } else {
      AnalyticsService.disable();
      AnalyticsService.deleteCollectedData();
    }
    if (!marketing) {
      PushNotificationService.unsubscribeMarketing();
    }
  }

  /// Call on app start — if consent version has changed, re-prompt the user
  bool get requiresConsentUpdate => !consentIsUpToDate;
}
```

**Consent screen rules:**
- Consent must be freely given, specific, informed, and unambiguous
- Pre-ticked boxes are not valid consent
- Declining must be as easy as accepting
- Re-request consent if the processing purposes change materially

## Right to Deletion

Users have the right to have their data erased. Implement a complete deletion flow:

```dart
class AccountDeletionService {
  final UserApiClient _api;
  final FlutterSecureStorage _secureStorage;
  final SharedPreferences _prefs;
  final LocalDatabase _db;

  Future<void> deleteAccount(String userId) async {
    // 1. Delete server-side data first
    await _api.deleteAccount(userId);   // API must cascade-delete all user records

    // 2. Clear all local data
    await _secureStorage.deleteAll();    // tokens, keys, cached credentials
    await _prefs.clear();               // preferences, consent records, cached profile
    await _db.deleteAllUserData();      // offline cache, synced records
    await _clearMediaCache();           // image cache, file downloads

    // 3. Log the deletion event (with timestamp, without PII)
    AuditLogger.log(
      event: 'account.deleted',
      properties: {'user_id_hash': _hash(userId)},
    );
  }

  Future<void> _clearMediaCache() async {
    final cacheDir = await getApplicationCacheDirectory();
    if (cacheDir.existsSync()) {
      cacheDir.deleteSync(recursive: true);
    }
  }
}
```

**Server-side deletion requirements:**
- Cascade delete across all tables and services that hold the user's data
- Remove from analytics platforms (Mixpanel, Amplitude have deletion APIs)
- Remove from email marketing lists
- Anonymize (not delete) records required for legal/accounting retention — replace PII with `[DELETED]` or a hash
- Respond within 30 days (GDPR) or 45 days (CCPA) of the request

## Data Export and Portability

Users have the right to receive their data in a portable format (GDPR Article 20):

```dart
class DataExportService {
  final UserApiClient _api;

  /// Request a data export — server generates and emails a download link
  Future<void> requestDataExport(String userId) async {
    await _api.requestDataExport(userId);
    // Server assembles: profile, activity history, preferences, uploaded content
    // Delivers as JSON or CSV archive to the user's registered email
  }

  /// In-app export of locally cached data
  Future<File> exportLocalData() async {
    final prefs   = await SharedPreferences.getInstance();
    final profile = await localDb.getUserProfile();

    final exportData = {
      'exported_at': DateTime.now().toIso8601String(),
      'preferences': prefs.getKeys().fold<Map<String, dynamic>>(
        {},
        (map, key) => map..[key] = prefs.get(key),
      ),
      'profile': profile?.toJson(),
    };

    final dir  = await getApplicationDocumentsDirectory();
    final file = File('${dir.path}/my_data_export.json');
    await file.writeAsString(jsonEncode(exportData));
    return file;
  }
}
```

## Code Obfuscation

Obfuscation renames classes, methods, and fields in the compiled binary, making static analysis and reverse engineering significantly harder.

```bash
# Android
flutter build apk --release \
  --obfuscate \
  --split-debug-info=build/symbols/android

flutter build appbundle --release \
  --obfuscate \
  --split-debug-info=build/symbols/android

# iOS
flutter build ipa --release \
  --obfuscate \
  --split-debug-info=build/symbols/ios

# Web (tree shaking and minification are applied by default with --release)
flutter build web --release
```

**`--split-debug-info`** writes symbol maps to the specified directory. Store these securely — you need them to deobfuscate crash stack traces from Crashlytics or Sentry.

**Deobfuscating a stack trace:**
```bash
flutter symbolize \
  --debug-info=build/symbols/android/app.android-arm64.symbols \
  --input=obfuscated_stack_trace.txt
```

**Limitations of obfuscation:**
- Does not encrypt your code
- Does not protect string literals (API keys, URLs remain readable — use `--dart-define`)
- Does not protect against dynamic analysis / runtime inspection
- Combine with root/jailbreak detection, certificate pinning, and proper key management

## Audit Logging

Maintain an audit trail of security-relevant events for compliance investigations and anomaly detection.

### What to Log

```dart
class AuditEvent {
  final String event;           // machine-readable event name
  final DateTime timestamp;
  final String? anonymousUserId; // hashed/pseudonymous — never the real user ID
  final String? sessionId;
  final Map<String, dynamic> properties; // event metadata — no PII

  const AuditEvent({
    required this.event,
    required this.timestamp,
    this.anonymousUserId,
    this.sessionId,
    this.properties = const {},
  });
}

// Events to log:
// auth.login_success, auth.login_failure, auth.logout
// auth.token_refresh, auth.token_expired
// account.password_changed, account.email_changed
// account.deletion_requested, account.deleted
// consent.updated, consent.withdrawn
// data.export_requested
// security.biometric_success, security.biometric_failure
// security.session_timeout
```

### What to Never Log

```dart
// NEVER log these:
// - Email addresses, phone numbers, names
// - Passwords (even hashed — log only success/failure events)
// - Access tokens or refresh tokens
// - Credit card numbers or payment details
// - Location coordinates
// - Device identifiers that link to a real person (IDFA, GAID)
// - Full request/response bodies from authenticated endpoints
// - Stack traces containing user input
```

### Implementation

```dart
class AuditLogger {
  static String _hashUserId(String userId) {
    // One-way hash — preserves ability to correlate events without storing real ID
    final bytes = utf8.encode(userId + _salt);
    return sha256.convert(bytes).toString().substring(0, 16);
  }

  static void log(
    String event, {
    String? userId,
    Map<String, dynamic> properties = const {},
  }) {
    final entry = AuditEvent(
      event: event,
      timestamp: DateTime.now().toUtc(),
      anonymousUserId: userId != null ? _hashUserId(userId) : null,
      sessionId: SessionManager.currentSessionId,
      properties: properties,
    );

    // Send to backend audit log service (separate from application logs)
    _sendToAuditService(entry);

    // In debug mode, also print locally
    assert(() {
      debugPrint('[AUDIT] ${entry.event} @ ${entry.timestamp.toIso8601String()}');
      return true;
    }());
  }
}
```

## App Store Privacy Declarations

### iOS Privacy Nutrition Labels (App Store Connect)

Declare all data types your app collects under Privacy → App Privacy:
- Data linked to identity: email, name, user ID (if you collect any of these)
- Usage data: product interaction, crash data
- Diagnostics: crash logs, performance data

For each type, declare whether it is linked to the user's identity and whether it is used for tracking.

### Google Play Data Safety Section

In Play Console → App content → Data safety, declare:
- Data collected (account info, location, messages, etc.)
- Whether each type is shared with third parties
- Whether it is required or optional
- Whether users can request deletion

Keep declarations synchronized with your actual code. Mismatches discovered in review or post-launch result in store removal.
