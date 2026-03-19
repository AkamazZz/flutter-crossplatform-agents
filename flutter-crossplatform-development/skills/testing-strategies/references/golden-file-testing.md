---
description: >
  Golden file testing setup, CI configuration, platform-specific considerations, and update workflows.
---

# Golden File Testing in Flutter

Golden file testing (snapshot testing) captures the pixel-accurate rendered output of a widget and stores it as a reference image. Subsequent test runs diff the live render against the stored file and fail if they diverge.

---

## Concept

A golden test answers: "Does this widget still look exactly like it did when I approved it?"

- On first run (or with `--update-goldens`): the reference PNG is written to disk.
- On every subsequent run: the widget is rendered and compared pixel-by-pixel against the stored PNG.
- Any change in layout, typography, color, or spacing fails the test.

Golden tests are valuable for:
- Design system components (buttons, cards, form fields)
- Screen layouts across multiple themes (light/dark) and text scales
- Locale-specific layouts (RTL, long strings)

---

## Setup

### Dependencies

```yaml
# pubspec.yaml
dev_dependencies:
  flutter_test:
    sdk: flutter
  golden_toolkit: ^0.15.0   # optional but recommended — adds font loading helpers
```

### Font Loading for Deterministic Renders

Different machines may have different system fonts, producing pixel differences that are not real regressions. Load the app's fonts explicitly before golden tests run.

```dart
// test/flutter_test_config.dart  (picked up automatically by the test runner)
import 'dart:async';

import 'package:flutter_test/flutter_test.dart';
import 'package:golden_toolkit/golden_toolkit.dart';

Future<void> testExecutable(FutureOr<void> Function() testMain) async {
  await loadAppFonts(); // loads fonts from pubspec.yaml assets
  return testMain();
}
```

If you are not using `golden_toolkit`, load fonts manually:

```dart
Future<void> setUpFonts() async {
  final fontLoader = FontLoader('Roboto')
    ..addFont(rootBundle.load('assets/fonts/Roboto-Regular.ttf'));
  await fontLoader.load();
}
```

---

## Writing a Golden Test

```dart
import 'package:flutter/material.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:golden_toolkit/golden_toolkit.dart';

void main() {
  group('PrimaryButton golden', () {
    testGoldens('light theme — default state', (tester) async {
      await tester.pumpWidgetBuilder(
        const PrimaryButton(label: 'Confirm'),
        wrapper: materialAppWrapper(theme: AppTheme.light()),
        surfaceSize: const Size(200, 60),
      );

      await screenMatchesGolden(tester, 'primary_button/light_default');
    });

    testGoldens('dark theme — loading state', (tester) async {
      await tester.pumpWidgetBuilder(
        const PrimaryButton(label: 'Confirm', isLoading: true),
        wrapper: materialAppWrapper(theme: AppTheme.dark()),
        surfaceSize: const Size(200, 60),
      );

      await screenMatchesGolden(tester, 'primary_button/dark_loading');
    });
  });
}
```

Without `golden_toolkit`, use the built-in matcher:

```dart
testWidgets('ProfileCard matches golden', (tester) async {
  await tester.pumpWidget(
    MaterialApp(
      theme: AppTheme.light(),
      home: Scaffold(body: ProfileCard(user: userFixture)),
    ),
  );

  await expectLater(
    find.byType(ProfileCard),
    matchesGoldenFile('goldens/profile_card_light.png'),
  );
});
```

---

## Creating and Updating Golden Files

```bash
# Generate (or regenerate) all golden files
flutter test --update-goldens

# Regenerate only specific test files
flutter test test/ui/profile_card_test.dart --update-goldens
```

Commit the generated PNG files to source control. Reviewers inspect the diff images visually in the PR.

---

## Naming Conventions

Use a hierarchical path that encodes the component, theme, state, and variant:

```
goldens/
  components/
    primary_button/
      light_default.png
      light_disabled.png
      dark_default.png
      dark_loading.png
  screens/
    login_page/
      light_empty.png
      light_error.png
      dark_empty.png
```

Pass the name without the `.png` extension to `matchesGoldenFile` or `screenMatchesGolden`; the framework appends it.

---

## CI Configuration

Golden tests that run on different OS produce slightly different pixel outputs due to font rasterisation differences. There are two strategies:

### Strategy 1 — Pin CI platform

Always run golden tests on the same OS. In GitHub Actions:

```yaml
# .github/workflows/golden_tests.yml
name: Golden Tests

on: [push, pull_request]

jobs:
  golden:
    runs-on: ubuntu-latest   # MUST match the platform used for --update-goldens

    steps:
      - uses: actions/checkout@v4

      - uses: subosito/flutter-action@v2
        with:
          flutter-version: '3.22.x'
          channel: stable

      - name: Install dependencies
        run: flutter pub get

      - name: Run golden tests
        run: flutter test --tags golden

      - name: Upload golden diffs on failure
        if: failure()
        uses: actions/upload-artifact@v4
        with:
          name: golden-failures
          path: test/failures/
```

### Strategy 2 — Platform-specific golden directories

Store separate reference images per platform:

```
goldens/
  linux/
    primary_button/light_default.png
  macos/
    primary_button/light_default.png
```

Resolve the platform at test time:

```dart
String goldenPath(String name) {
  final platform = Platform.isLinux ? 'linux'
      : Platform.isMacOS ? 'macos'
      : 'windows';
  return 'goldens/$platform/$name.png';
}

await expectLater(
  find.byType(PrimaryButton),
  matchesGoldenFile(goldenPath('primary_button/light_default')),
);
```

---

## Updating Goldens vs Real Regressions

Ask these questions before running `--update-goldens`:

- Designer intentionally changed the UI → update goldens, attach before/after in PR
- Dependency upgrade changed rendering → investigate; if correct, update with explanation
- Refactor changed layout unexpectedly → regression; fix the code, do NOT update goldens
- Font/platform rendering noise difference → use platform pinning or per-platform goldens

Never run `--update-goldens` to silence a failure without understanding why the diff occurred.

---

## Recommended Tags

Tag golden tests so they can be run independently from unit/widget tests:

```dart
// In the test file
@Tags(['golden'])
library;

testGoldens('...', (tester) async { ... });
```

```bash
# Run only golden tests
flutter test --tags golden

# Exclude golden tests from the fast test suite
flutter test --exclude-tags golden
```
