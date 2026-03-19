---
name: security-compliance
description: >
  Flutter security best practices including secure storage, network security, and compliance.
  Use when implementing authentication, securing data, hardening the app, or ensuring regulatory
  compliance. Trigger on: secure storage, Keychain, KeyStore, certificate pinning, biometric
  auth, GDPR, obfuscation, encryption, API security, tokens.
---

# Security & Compliance

Flutter app security operates across four layers: **storage → network → app hardening → compliance**. Each layer has distinct threats and mitigations, and weaknesses at any layer can undermine the others. Address all four systematically rather than treating security as a feature to add later.

## Security Layer Overview

```
[Storage Layer]
  Sensitive data (tokens, keys, PII) → flutter_secure_storage → iOS Keychain / Android KeyStore
  Non-sensitive preferences → shared_preferences
  Encryption at rest for local databases (SQLCipher)

      │
      ▼

[Network Layer]
  All traffic over HTTPS/TLS 1.2+
  Certificate pinning via Dio interceptor (SHA-256)
  API keys never in source — injected via --dart-define
  Request signing (HMAC) for sensitive endpoints
  Android network_security_config.xml + iOS ATS

      │
      ▼

[App Hardening Layer]
  Code obfuscation: --obfuscate --split-debug-info
  Root/jailbreak detection
  Screenshot prevention on sensitive screens
  Biometric authentication (local_auth)
  Session timeout + token rotation

      │
      ▼

[Compliance Layer]
  GDPR: lawful basis, consent, right to deletion, data portability
  Privacy-by-design: collect minimum data, no PII in logs
  Audit logging: structured, anonymized event logs
  App store privacy declarations (iOS nutrition labels, Play Data Safety)
```

## Reference Documents

| Document | Contents | When to Use |
|---|---|---|
| [secure-storage.md](references/secure-storage.md) | flutter_secure_storage setup, token lifecycle, iOS Keychain, Android KeyStore, decision guide | Storing credentials, tokens, or any sensitive user data |
| [network-security.md](references/network-security.md) | Certificate pinning, HTTPS enforcement, API key management, request signing, platform security configs | Securing API communication and preventing interception |
| [compliance-gdpr.md](references/compliance-gdpr.md) | GDPR principles, consent management, right to deletion, obfuscation, audit logging | Building privacy-compliant features or preparing for regulatory review |

## Quick Reference

**Secure storage read/write:**
```dart
final storage = FlutterSecureStorage(
  aOptions: AndroidOptions(encryptedSharedPreferences: true),
  iOptions: IOSOptions(accessibility: KeychainAccessibility.first_unlock),
);
await storage.write(key: 'access_token', value: token);
final token = await storage.read(key: 'access_token');
```

**Build with obfuscation:**
```bash
flutter build apk --release \
  --obfuscate \
  --split-debug-info=build/symbols/android
```

**Inject API keys — never hardcode:**
```bash
flutter build apk --dart-define=API_KEY=secret_value
```
```dart
const apiKey = String.fromEnvironment('API_KEY'); // empty string in debug if not set
```

**Certificate pinning (Dio):**
```dart
(dio.httpClientAdapter as IOHttpClientAdapter).createHttpClient = () {
  final client = HttpClient();
  client.badCertificateCallback = (cert, host, port) => false;
  // See network-security.md for full SHA-256 pinning implementation
  return client;
};
```

**Biometric gate:**
```dart
final auth = LocalAuthentication();
final authenticated = await auth.authenticate(
  localizedReason: 'Confirm your identity to continue',
  options: const AuthenticationOptions(biometricOnly: false),
);
```

## Anti-Patterns

- **Storing tokens in SharedPreferences** — always use `flutter_secure_storage` for access tokens, refresh tokens, and any credentials; SharedPreferences is unencrypted plain text on Android
- **Hardcoding API keys in source** — keys in Dart source end up in the compiled binary and can be extracted; use `--dart-define` and rotate compromised keys immediately
- **Logging PII or tokens** — never log email addresses, names, tokens, or device identifiers; use anonymized event names and IDs only
- **Skipping certificate pinning** — without pinning, a compromised CA or MITM proxy can intercept all HTTPS traffic transparently
- **No token expiry or rotation** — long-lived tokens that never rotate are a persistent breach risk; implement access + refresh token patterns with short access token lifetimes
- **Obfuscation as the only protection** — obfuscation raises the bar but is not encryption; combine it with proper key management, certificate pinning, and secure storage

