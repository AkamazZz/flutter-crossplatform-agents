---
description: >
  Integration testing with Patrol, E2E test patterns, device farm configuration, and CI integration.
---

# Integration Testing in Flutter

Integration tests run the full compiled app on a real or emulated device. They test complete user journeys, native platform interactions, and cross-feature flows that cannot be verified in widget tests.

---

## `integration_test` Package (Flutter Native)

The `integration_test` package ships with the Flutter SDK. It hooks `testWidgets` into the device's test runner.

### Setup

```yaml
# pubspec.yaml
dev_dependencies:
  integration_test:
    sdk: flutter
  flutter_test:
    sdk: flutter
```

### Test entry point

```dart
// integration_test/app_test.dart
import 'package:flutter_test/flutter_test.dart';
import 'package:integration_test/integration_test.dart';

import 'package:my_app/main.dart' as app;

void main() {
  IntegrationTestWidgetsFlutterBinding.ensureInitialized();

  group('end-to-end smoke tests', () {
    testWidgets('user can sign in and reach home screen', (tester) async {
      app.main();
      await tester.pumpAndSettle();

      await tester.enterText(find.byKey(const Key('email-field')), 'test@example.com');
      await tester.enterText(find.byKey(const Key('password-field')), 'password');
      await tester.tap(find.byKey(const Key('submit-button')));
      await tester.pumpAndSettle();

      expect(find.byType(HomeScreen), findsOneWidget);
    });
  });
}
```

### Running on device / emulator

```bash
# Run on connected device
flutter test integration_test/app_test.dart

# Run on a specific device
flutter test integration_test/app_test.dart -d emulator-5554
```

---

## Patrol — Native Interaction Layer

[Patrol](https://patrol.leancode.co) wraps `integration_test` with a `PatrolTester` that can interact with native OS elements: system dialogs, permission prompts, notifications, and deep links.

### Setup

```yaml
# pubspec.yaml
dev_dependencies:
  patrol: ^3.0.0
  integration_test:
    sdk: flutter
```

```yaml
# patrol.yaml
app_name: MyApp
android:
  app_id: com.example.myapp
  package_name: com.example.myapp
ios:
  bundle_id: com.example.myapp
```

Install the Patrol CLI:

```bash
dart pub global activate patrol_cli
```

### `PatrolTester` — basic test structure

```dart
import 'package:patrol/patrol.dart';
import 'package:my_app/main.dart' as app;

void main() {
  patrolTest(
    'user grants camera permission and takes photo',
    ($) async {
      app.main();
      await $.pumpAndSettle();

      await $(CameraButton).tap();

      // Handle the native iOS/Android permission dialog
      if (await $.native.isPermissionDialogVisible()) {
        await $.native.grantPermissionWhenInUse();
      }

      await $.pumpAndSettle();
      expect($(PhotoPreview), findsOneWidget);
    },
  );
}
```

### Native interactions

```dart
// Tap "Allow" on a system permission dialog
await $.native.grantPermissionWhenInUse();     // location / camera while in use
await $.native.grantPermissionOnlyThisTime();  // one-time grant
await $.native.denyPermission();               // deny

// Open the notification shade and tap a notification
await $.native.openNotifications();
await $.native.tapOnNotificationByTitle('New message');

// Handle a URL / deep link
await $.native.openApp(appId: 'com.example.myapp');

// Enter text in a native text field (e.g. system keyboard prompt)
await $.native.enterTextByIndex('secret', index: 0);
```

### Running Patrol tests

```bash
# Android
patrol test --target integration_test/app_test.dart

# iOS (requires connected device or simulator)
patrol test --target integration_test/app_test.dart --platform ios
```

---

## Test Drivers for CI

The `integration_test` package requires a driver to run on CI without interactive setup.

```dart
// test_driver/integration_test.dart
import 'package:integration_test/integration_test_driver.dart';

Future<void> main() => integrationDriver();
```

Running via driver:

```bash
flutter drive \
  --driver=test_driver/integration_test.dart \
  --target=integration_test/app_test.dart \
  -d emulator-5554
```

---

## Device Farms

### Firebase Test Lab

```bash
# Build the APK for instrumentation
flutter build apk --debug

# Build the test APK
./gradlew app:assembleAndroidTest

# Submit to Firebase Test Lab
gcloud firebase test android run \
  --type instrumentation \
  --app build/app/outputs/flutter-apk/app-debug.apk \
  --test build/app/outputs/apk/androidTest/debug/app-debug-androidTest.apk \
  --device model=Pixel6,version=33,locale=en,orientation=portrait
```

For iOS on Firebase Test Lab, use XCTest bundles:

```bash
flutter build ios --release --no-codesign
cd ios && xcodebuild build-for-testing \
  -workspace Runner.xcworkspace \
  -scheme Runner \
  FLUTTER_BUILD_MODE=release \
  -derivedDataPath ../build/ios_integ
```

### AWS Device Farm

Upload `.ipa` / `.apk` and the test package through the AWS Console or CLI. Use the `appium_flutter_driver` or the native `XCTest` / `Instrumentation` test type depending on the test framework.

---

## GitHub Actions Workflow

```yaml
# .github/workflows/integration_tests.yml
name: Integration Tests

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  integration-android:
    runs-on: ubuntu-latest
    timeout-minutes: 30

    steps:
      - uses: actions/checkout@v4

      - name: Set up Java
        uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: temurin

      - uses: subosito/flutter-action@v2
        with:
          flutter-version: '3.22.x'
          channel: stable

      - name: Install dependencies
        run: flutter pub get

      - name: Enable KVM (Android emulator acceleration)
        run: |
          echo 'KERNEL=="kvm", GROUP="kvm", MODE="0666", OPTIONS+="static_node=kvm"' \
            | sudo tee /etc/udev/rules.d/99-kvm4all.rules
          sudo udevadm control --reload-rules
          sudo udevadm trigger --name-match=kvm

      - name: Launch Android emulator
        uses: reactivecircus/android-emulator-runner@v2
        with:
          api-level: 33
          target: google_apis
          arch: x86_64
          script: flutter test integration_test/app_test.dart -d emulator-5554

  integration-ios:
    runs-on: macos-14
    timeout-minutes: 45

    steps:
      - uses: actions/checkout@v4

      - uses: subosito/flutter-action@v2
        with:
          flutter-version: '3.22.x'
          channel: stable

      - name: Install dependencies
        run: flutter pub get

      - name: Boot iOS Simulator
        run: |
          UDID=$(xcrun simctl create "iPhone 15" "com.apple.CoreSimulator.SimDeviceType.iPhone-15" \
            "com.apple.CoreSimulator.SimRuntime.iOS-17-0")
          xcrun simctl boot "$UDID"
          echo "SIMULATOR_UDID=$UDID" >> $GITHUB_ENV

      - name: Run integration tests
        run: flutter test integration_test/app_test.dart -d "$SIMULATOR_UDID"

      - name: Shutdown simulator
        if: always()
        run: xcrun simctl shutdown "$SIMULATOR_UDID"
```

---

## When Integration Tests vs Widget Tests

- Verifying a BLoC emits the correct states → Unit test
- A single widget renders its sealed states → Widget test
- A screen navigates correctly between routes (mocked BLoC) → Widget test
- An end-to-end flow across multiple screens with real data → Integration test
- Native permission dialog must be accepted → Integration (Patrol)
- Push notification opens the correct screen → Integration (Patrol)
- Camera / biometrics / Bluetooth interaction → Integration (Patrol)
- Visual regression of a design system component → Golden file test

Integration tests are slow and fragile. Write them only for critical user journeys and native interactions that cannot be simulated in widget tests. For everything else, prefer unit and widget tests.

---

## Tips for Stable Integration Tests

- **Use semantic keys** (`Key('submit-button')`) on interactive widgets; text labels change, keys should not.
- **Avoid fixed `sleep` calls**; prefer `pumpAndSettle` or `waitUntilVisible`.
- **Reset app state** between tests — use a test-only `resetDatabase()` endpoint or dependency injection overrides.
- **Run on a fixed device spec** in CI to avoid flakiness from differing screen densities.
- **Retry flaky tests once** before marking them as failures; native timing issues are common.
- **Tag integration tests** (`@Tags(['integration'])`) and exclude them from the fast test suite:

```bash
# Fast suite (excludes integration)
flutter test --exclude-tags integration

# Full suite
flutter test
```
