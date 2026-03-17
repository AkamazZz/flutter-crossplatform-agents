---
description: >
  Certificate pinning, TLS configuration, API security, and network hardening best practices.
---

# Network Security

Every byte leaving the device is a potential attack surface. Enforce TLS, pin certificates to prevent interception, manage API keys out-of-band, and configure platform-level security policies on both Android and iOS.

## HTTPS Enforcement

Never allow plain HTTP in production. Enforce at multiple levels:

**Dart / Dio — reject non-HTTPS URLs:**
```dart
class HttpsEnforcementInterceptor extends Interceptor {
  @override
  void onRequest(RequestOptions options, RequestInterceptorHandler handler) {
    if (!options.uri.isScheme('https')) {
      handler.reject(
        DioException(
          requestOptions: options,
          message: 'Non-HTTPS requests are not permitted.',
        ),
        true,
      );
      return;
    }
    super.onRequest(options, handler);
  }
}
```

**Minimum TLS version (Dio):**
```dart
final dio = Dio();
(dio.httpClientAdapter as IOHttpClientAdapter).createHttpClient = () {
  final client = HttpClient();
  client.minTlsProtocolVersion = TlsProtocolVersion.tlsV12;
  return client;
};
```

## Certificate Pinning

Certificate pinning ensures your app only communicates with servers presenting a known certificate, preventing interception by rogue CAs or MITM proxies even over valid HTTPS.

### SHA-256 Public Key Pinning with Dio

```dart
import 'dart:convert';
import 'dart:io';
import 'package:crypto/crypto.dart';
import 'package:dio/dio.dart';
import 'package:dio/io.dart';

class CertificatePinningInterceptor {
  // Extract with: openssl s_client -connect api.example.com:443 |
  //   openssl x509 -pubkey -noout | openssl pkey -pubin -outform DER |
  //   openssl dgst -sha256 -binary | base64
  static const _pinnedKeys = <String>{
    'AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA=',  // primary cert
    'BBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBB=',  // backup cert
  };

  static void apply(Dio dio) {
    (dio.httpClientAdapter as IOHttpClientAdapter).createHttpClient = () {
      final client = HttpClient();
      client.badCertificateCallback = (X509Certificate cert, String host, int port) {
        // Extract the public key SHA-256 fingerprint
        final fingerprint = _getPublicKeyFingerprint(cert);
        if (_pinnedKeys.contains(fingerprint)) {
          return false; // certificate is valid (no error)
        }
        // Reject the connection — fingerprint does not match
        return true;   // returning true would allow bad certs — DO NOT do this
      };
      return client;
    };
  }

  static String _getPublicKeyFingerprint(X509Certificate cert) {
    // cert.der contains the raw DER-encoded certificate
    // For public key pinning, extract the SubjectPublicKeyInfo bytes
    // This simplified version pins the full certificate DER hash:
    final digest = sha256.convert(cert.der);
    return base64.encode(digest.bytes);
  }
}
```

**Apply to your Dio instance:**
```dart
final dio = Dio(BaseOptions(baseUrl: 'https://api.example.com'));
CertificatePinningInterceptor.apply(dio);
```

### Obtaining the PIN

```bash
# Get the SHA-256 public key pin for your server
openssl s_client -connect api.example.com:443 -servername api.example.com < /dev/null 2>/dev/null \
  | openssl x509 -pubkey -noout \
  | openssl pkey -pubin -outform DER \
  | openssl dgst -sha256 -binary \
  | base64
```

**Always pin at least two certificates** (primary + backup/intermediate) to allow key rotation without a forced app update.

### Certificate Rotation Strategy

1. Generate new certificate with ample lead time (weeks before expiry)
2. Add the new certificate's pin to the app and ship an update
3. Wait for the majority of users to update (track via analytics)
4. Rotate the server certificate
5. Remove the old pin in a subsequent release

## API Key Management

API keys embedded in source code or compiled binaries can be extracted with `strings`, decompilers, or MITM proxies. Never hardcode them.

**Correct approach — inject at build time:**
```bash
flutter build apk \
  --dart-define=MAPS_API_KEY=AIzaSy... \
  --dart-define=ANALYTICS_KEY=abc123...
```

**Access in code:**
```dart
// Compile-time constant — not accessible at runtime by reflection
const mapsApiKey     = String.fromEnvironment('MAPS_API_KEY');
const analyticsKey   = String.fromEnvironment('ANALYTICS_KEY');
```

**For CI — store in secrets, never in the repo:**
```yaml
# GitHub Actions
- name: Build
  run: flutter build apk --dart-define=MAPS_API_KEY=${{ secrets.MAPS_API_KEY }}
```

**Additional protections:**
- Restrict API keys by bundle ID / package name in the provider's console (Google Cloud, Firebase, etc.)
- Restrict by IP address for server-side keys
- Rotate keys periodically and immediately if a leak is suspected
- Use a proxy/BFF (Backend for Frontend) server to hold secrets server-side when possible

## Request Signing with HMAC

For high-value endpoints (payment, account changes), sign requests so the server can verify they originate from your app and have not been tampered with.

```dart
import 'dart:convert';
import 'package:crypto/crypto.dart';

class HmacSigningInterceptor extends Interceptor {
  final String _secretKey;

  HmacSigningInterceptor(this._secretKey);

  @override
  void onRequest(RequestOptions options, RequestInterceptorHandler handler) {
    final timestamp = DateTime.now().millisecondsSinceEpoch.toString();
    final nonce     = _generateNonce();
    final body      = options.data != null ? jsonEncode(options.data) : '';

    final message = '${options.method.toUpperCase()}\n'
        '${options.path}\n'
        '$timestamp\n'
        '$nonce\n'
        '$body';

    final hmac      = Hmac(sha256, utf8.encode(_secretKey));
    final digest    = hmac.convert(utf8.encode(message));
    final signature = base64.encode(digest.bytes);

    options.headers['X-Timestamp'] = timestamp;
    options.headers['X-Nonce']     = nonce;
    options.headers['X-Signature'] = signature;

    handler.next(options);
  }

  String _generateNonce() =>
      base64Url.encode(List<int>.generate(16, (_) => Random.secure().nextInt(256)));
}
```

The server validates the signature using the shared secret, rejects replays using the timestamp window (e.g., ±5 minutes), and checks nonce uniqueness.

## Android network_security_config.xml

Configure Android's Network Security Configuration to block cleartext traffic and optionally pin certificates at the OS level:

```xml
<!-- android/app/src/main/res/xml/network_security_config.xml -->
<?xml version="1.0" encoding="utf-8"?>
<network-security-config>
    <!-- Block all cleartext (HTTP) traffic -->
    <base-config cleartextTrafficPermitted="false">
        <trust-anchors>
            <certificates src="system" />
        </trust-anchors>
    </base-config>

    <!-- Allow cleartext only for local development -->
    <domain-config cleartextTrafficPermitted="true">
        <domain includeSubdomains="true">localhost</domain>
        <domain includeSubdomains="true">10.0.2.2</domain>
    </domain-config>

    <!-- Certificate pinning at OS level (backup to Dart-level pinning) -->
    <domain-config>
        <domain includeSubdomains="true">api.example.com</domain>
        <pin-set expiration="2026-12-31">
            <pin digest="SHA-256">AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA=</pin>
            <pin digest="SHA-256">BBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBB=</pin>
        </pin-set>
    </domain-config>
</network-security-config>
```

Reference in AndroidManifest.xml:
```xml
<application
    android:networkSecurityConfig="@xml/network_security_config"
    ...>
```

## iOS App Transport Security

iOS ATS enforces HTTPS by default. Configure exceptions carefully in `Info.plist`:

```xml
<!-- ios/Runner/Info.plist -->
<key>NSAppTransportSecurity</key>
<dict>
    <!-- Allow all connections (DO NOT use in production) -->
    <!-- <key>NSAllowsArbitraryLoads</key><true/> -->

    <!-- Recommended: domain-specific exceptions only -->
    <key>NSExceptionDomains</key>
    <dict>
        <!-- Allow HTTP only for local development -->
        <key>localhost</key>
        <dict>
            <key>NSExceptionAllowsInsecureHTTPLoads</key>
            <true/>
        </dict>
    </dict>
</dict>
```

For production, **do not set `NSAllowsArbitraryLoads: true`**. App Store review flags this and it weakens security for all domains. If you need an exception for a third-party SDK, use `NSExceptionDomains` with the minimum required permissions.

**Certificate transparency** (iOS 10+, enabled by default): ATS requires certificates to be logged in Certificate Transparency logs. Do not disable this.

## Security Headers

Validate that your API server sends these headers and that your Dio client respects them:

```dart
class SecurityHeadersInterceptor extends Interceptor {
  @override
  void onResponse(Response response, ResponseInterceptorHandler handler) {
    // Log warnings if security headers are missing (don't block, but alert)
    final headers = response.headers;
    if (headers.value('strict-transport-security') == null) {
      debugPrint('Warning: HSTS header missing from ${response.requestOptions.uri.host}');
    }
    if (headers.value('x-content-type-options') == null) {
      debugPrint('Warning: X-Content-Type-Options header missing');
    }
    handler.next(response);
  }
}
```

Expected security headers on your API:
- `Strict-Transport-Security: max-age=31536000; includeSubDomains`
- `X-Content-Type-Options: nosniff`
- `X-Frame-Options: DENY`
- `Content-Security-Policy` (for web)
- `Cache-Control: no-store` on authentication endpoints
