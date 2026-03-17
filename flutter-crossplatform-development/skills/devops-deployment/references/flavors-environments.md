---
description: >
  Flutter flavors, environment-specific configurations, and build variant management.
---

# Flavors & Environments

Flutter flavors let you maintain separate builds — dev, staging, and prod — from a single codebase. Each flavor can have its own bundle ID, app name, icons, API endpoint, and feature flags, all resolved at build time with no runtime switching required.

## Flavor Entry Points

Create a separate `main_*.dart` per flavor:

```
lib/
  main_dev.dart
  main_staging.dart
  main_prod.dart
  app_config.dart
```

```dart
// lib/main_dev.dart
import 'package:flutter/material.dart';
import 'app_config.dart';
import 'main.dart' as app;

void main() {
  AppConfig.initialize(
    flavor: Flavor.dev,
    apiBaseUrl: 'https://dev.api.example.com',
    sentryDsn: '',
  );
  app.main();
}
```

```dart
// lib/app_config.dart
enum Flavor { dev, staging, prod }

class AppConfig {
  static late Flavor flavor;
  static late String apiBaseUrl;
  static late String sentryDsn;

  static void initialize({
    required Flavor flavor,
    required String apiBaseUrl,
    required String sentryDsn,
  }) {
    AppConfig.flavor = flavor;
    AppConfig.apiBaseUrl = apiBaseUrl;
    AppConfig.sentryDsn = sentryDsn;
  }

  static bool get isDev => flavor == Flavor.dev;
  static bool get isProd => flavor == Flavor.prod;
}
```

## --dart-define and --dart-define-from-file

Inject values at compile time. They are baked into the binary and cannot be changed at runtime.

**Single values:**
```bash
flutter run \
  --flavor dev \
  --target lib/main_dev.dart \
  --dart-define=API_URL=https://dev.api.example.com \
  --dart-define=MAPS_KEY=AIzaSy...
```

**From a file (Flutter 3.7+, recommended):**
```bash
flutter build apk \
  --flavor prod \
  --target lib/main_prod.dart \
  --dart-define-from-file=config/prod.json
```

```json
// config/prod.json  (do NOT commit files with real secrets)
{
  "API_URL": "https://api.example.com",
  "MAPS_KEY": "AIzaSy...",
  "SENTRY_DSN": "https://..."
}
```

```dart
// Reading in Dart
const apiUrl    = String.fromEnvironment('API_URL',  defaultValue: 'http://localhost:8080');
const mapsKey   = String.fromEnvironment('MAPS_KEY', defaultValue: '');
const sentryDsn = String.fromEnvironment('SENTRY_DSN', defaultValue: '');
```

> Keep `config/prod.json` in `.gitignore`. Pass real values via CI environment variables at build time.

## Per-Flavor Bundle IDs, App Names, and Icons

### Android — build.gradle (app/build.gradle)

```groovy
android {
    flavorDimensions "environment"

    productFlavors {
        dev {
            dimension "environment"
            applicationId "com.example.app.dev"
            resValue "string", "app_name", "MyApp Dev"
            versionNameSuffix "-dev"
        }
        staging {
            dimension "environment"
            applicationId "com.example.app.staging"
            resValue "string", "app_name", "MyApp Staging"
            versionNameSuffix "-staging"
        }
        prod {
            dimension "environment"
            applicationId "com.example.app"
            resValue "string", "app_name", "MyApp"
        }
    }
}
```

Per-flavor resource directories:
```
android/app/src/
  dev/res/mipmap-*/ic_launcher.png
  staging/res/mipmap-*/ic_launcher.png
  prod/res/mipmap-*/ic_launcher.png   (or main/ for the default)
```

### iOS — Schemes and Configurations in Xcode

1. In Xcode, open **Product > Scheme > Manage Schemes**, duplicate the default scheme for each flavor (e.g., `Runner-Dev`, `Runner-Staging`, `Runner-Prod`).
2. Add Build Configurations: **Debug-Dev**, **Release-Dev**, **Debug-Staging**, **Release-Staging**, **Debug-Prod**, **Release-Prod**.
3. Set the bundle identifier per configuration in **Signing & Capabilities**:

```
// In Runner/Info.plist or per-config xcconfig files:
// Debug-Dev:     PRODUCT_BUNDLE_IDENTIFIER = com.example.app.dev
// Release-Staging: PRODUCT_BUNDLE_IDENTIFIER = com.example.app.staging
// Release-Prod:  PRODUCT_BUNDLE_IDENTIFIER = com.example.app
```

4. Set `FLUTTER_TARGET` in each scheme's Build pre-action or pass `--target` via CLI.

Per-flavor app name in `Info.plist`:
```xml
<key>CFBundleDisplayName</key>
<string>$(APP_DISPLAY_NAME)</string>
```
Define `APP_DISPLAY_NAME` in each `.xcconfig` file.

## Per-Flavor App Icons with flutter_launcher_icons

```yaml
# pubspec.yaml
dev_dependencies:
  flutter_launcher_icons: ^0.13.0

flutter_launcher_icons:
  android: true
  ios: true
  image_path: "assets/icons/icon_prod.png"
```

For per-flavor icons, create separate config files:

```yaml
# flutter_launcher_icons-dev.yaml
flutter_launcher_icons:
  android: "ic_launcher"
  ios: true
  image_path: "assets/icons/icon_dev.png"
  adaptive_icon_background: "#FF5722"
  adaptive_icon_foreground: "assets/icons/icon_dev_fg.png"
```

```bash
# Generate icons for each flavor
dart run flutter_launcher_icons -f flutter_launcher_icons-dev.yaml
dart run flutter_launcher_icons -f flutter_launcher_icons-staging.yaml
dart run flutter_launcher_icons -f flutter_launcher_icons-prod.yaml
```

## Environment-Specific API Endpoints

Centralize endpoint resolution so it cannot be confused:

```dart
class ApiEndpoints {
  static const _base = String.fromEnvironment(
    'API_URL',
    defaultValue: 'https://dev.api.example.com',
  );

  static String get users     => '$_base/users';
  static String get auth      => '$_base/auth';
  static String get products  => '$_base/products';
}
```

For local development with a device on the same network:
```bash
flutter run \
  --flavor dev \
  --target lib/main_dev.dart \
  --dart-define=API_URL=http://192.168.1.100:8080
```

## Running and Building Flavors

```bash
# Development
flutter run   --flavor dev     --target lib/main_dev.dart
flutter build apk --flavor dev --target lib/main_dev.dart --debug

# Staging
flutter run   --flavor staging --target lib/main_staging.dart
flutter build apk --flavor staging --target lib/main_staging.dart

# Production
flutter build apk --flavor prod --target lib/main_prod.dart --release
flutter build appbundle --flavor prod --target lib/main_prod.dart --release
flutter build ipa --flavor prod --target lib/main_prod.dart --release
```
