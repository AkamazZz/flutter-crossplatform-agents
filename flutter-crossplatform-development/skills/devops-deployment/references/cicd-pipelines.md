---
description: >
  CI/CD pipeline setup with GitHub Actions, Codemagic, and automated testing workflows.
---

# CI/CD Pipelines

Automated pipelines eliminate manual build steps, enforce quality gates, and deliver signed artifacts consistently. This document covers GitHub Actions and Codemagic configurations with caching, testing, versioning, and artifact publishing.

## Branch Strategy

```
main ──────────────────────────────────────────── production releases
  │
  ├── develop ───────────────────────────────────── staging releases
  │     │
  │     ├── feature/auth-screen
  │     ├── feature/payment-flow
  │     └── fix/login-crash
  │
  └── release/1.2.0 ──────────────────────────── release candidate
```

- `feature/*`, `fix/*` — PR to develop → run tests only
- `develop` — push → build staging APK + IPA
- `release/*` — push → build prod AAB + IPA, upload to stores
- `main` — push (merge) → tag release, create GitHub Release

## GitHub Actions

### Complete Workflow

```yaml
# .github/workflows/flutter_ci.yml
name: Flutter CI/CD

on:
  push:
    branches: [main, develop, 'release/**']
  pull_request:
    branches: [main, develop]

env:
  FLUTTER_VERSION: '3.22.0'

jobs:
  # ─── Analyze & Test ──────────────────────────────────────────
  test:
    name: Analyze & Test
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup Flutter
        uses: subosito/flutter-action@v2
        with:
          flutter-version: ${{ env.FLUTTER_VERSION }}
          channel: stable
          cache: true                        # caches Flutter SDK

      - name: Cache pub dependencies
        uses: actions/cache@v4
        with:
          path: |
            ~/.pub-cache
            .dart_tool
          key: pub-${{ runner.os }}-${{ hashFiles('**/pubspec.lock') }}
          restore-keys: |
            pub-${{ runner.os }}-

      - name: Install dependencies
        run: flutter pub get

      - name: Analyze
        run: flutter analyze --fatal-infos

      - name: Run tests
        run: flutter test --coverage --reporter=github

      - name: Upload coverage
        uses: codecov/codecov-action@v4
        with:
          file: coverage/lcov.info
          token: ${{ secrets.CODECOV_TOKEN }}

  # ─── Build Android ───────────────────────────────────────────
  build-android:
    name: Build Android
    needs: test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/develop' || startsWith(github.ref, 'refs/heads/release/')

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup Java
        uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: '17'

      - name: Setup Flutter
        uses: subosito/flutter-action@v2
        with:
          flutter-version: ${{ env.FLUTTER_VERSION }}
          channel: stable
          cache: true

      - name: Cache pub dependencies
        uses: actions/cache@v4
        with:
          path: |
            ~/.pub-cache
            .dart_tool
          key: pub-${{ runner.os }}-${{ hashFiles('**/pubspec.lock') }}
          restore-keys: pub-${{ runner.os }}-

      - name: Cache Gradle
        uses: actions/cache@v4
        with:
          path: |
            ~/.gradle/caches
            ~/.gradle/wrapper
          key: gradle-${{ runner.os }}-${{ hashFiles('**/*.gradle*', '**/gradle-wrapper.properties') }}

      - name: Decode keystore
        run: |
          echo "${{ secrets.KEYSTORE_BASE64 }}" | base64 --decode > android/app/keystore.jks

      - name: Set version
        run: |
          VERSION_NAME=$(grep 'version:' pubspec.yaml | sed 's/version: //' | cut -d'+' -f1)
          BUILD_NUMBER=${{ github.run_number }}
          flutter build appbundle \
            --flavor prod \
            --target lib/main_prod.dart \
            --release \
            --build-name=$VERSION_NAME \
            --build-number=$BUILD_NUMBER \
            --dart-define=API_URL=${{ secrets.PROD_API_URL }}
        env:
          KEYSTORE_PATH: keystore.jks
          KEYSTORE_PASSWORD: ${{ secrets.KEYSTORE_PASSWORD }}
          KEY_ALIAS: ${{ secrets.KEY_ALIAS }}
          KEY_PASSWORD: ${{ secrets.KEY_PASSWORD }}

      - name: Upload AAB artifact
        uses: actions/upload-artifact@v4
        with:
          name: android-release-aab
          path: build/app/outputs/bundle/prodRelease/app-prod-release.aab
          retention-days: 7

  # ─── Build iOS ───────────────────────────────────────────────
  build-ios:
    name: Build iOS
    needs: test
    runs-on: macos-14
    if: github.ref == 'refs/heads/develop' || startsWith(github.ref, 'refs/heads/release/')

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup Flutter
        uses: subosito/flutter-action@v2
        with:
          flutter-version: ${{ env.FLUTTER_VERSION }}
          channel: stable
          cache: true

      - name: Cache pub dependencies
        uses: actions/cache@v4
        with:
          path: |
            ~/.pub-cache
            .dart_tool
          key: pub-${{ runner.os }}-${{ hashFiles('**/pubspec.lock') }}
          restore-keys: pub-${{ runner.os }}-

      - name: Import certificates
        uses: apple-actions/import-codesign-certs@v3
        with:
          p12-file-base64: ${{ secrets.IOS_DIST_CERT_P12_BASE64 }}
          p12-password: ${{ secrets.IOS_DIST_CERT_PASSWORD }}

      - name: Download provisioning profile
        run: |
          mkdir -p ~/Library/MobileDevice/Provisioning\ Profiles
          echo "${{ secrets.IOS_PROVISIONING_PROFILE_BASE64 }}" | base64 --decode \
            > ~/Library/MobileDevice/Provisioning\ Profiles/prod.mobileprovision

      - name: Build IPA
        run: |
          flutter build ipa \
            --flavor prod \
            --target lib/main_prod.dart \
            --release \
            --export-options-plist=ios/ExportOptions.plist \
            --dart-define=API_URL=${{ secrets.PROD_API_URL }}

      - name: Upload IPA artifact
        uses: actions/upload-artifact@v4
        with:
          name: ios-release-ipa
          path: build/ios/ipa/*.ipa
          retention-days: 7

  # ─── Deploy (release branches only) ──────────────────────────
  deploy:
    name: Deploy to Stores
    needs: [build-android, build-ios]
    runs-on: ubuntu-latest
    if: startsWith(github.ref, 'refs/heads/release/')

    steps:
      - name: Download Android artifact
        uses: actions/download-artifact@v4
        with:
          name: android-release-aab

      - name: Upload to Play Store (Internal Track)
        uses: r0adkll/upload-google-play@v1
        with:
          serviceAccountJsonPlainText: ${{ secrets.PLAY_SERVICE_ACCOUNT_JSON }}
          packageName: com.example.app
          releaseFiles: '*.aab'
          track: internal

      # iOS upload via Fastlane or xcrun altool (run on macOS runner)
```

### Required GitHub Secrets

- `KEYSTORE_BASE64` — base64-encoded Android keystore `.jks` file
- `KEYSTORE_PASSWORD` — keystore password
- `KEY_ALIAS` — key alias within the keystore
- `KEY_PASSWORD` — key password
- `IOS_DIST_CERT_P12_BASE64` — base64-encoded iOS distribution certificate `.p12`
- `IOS_DIST_CERT_PASSWORD` — certificate password
- `IOS_PROVISIONING_PROFILE_BASE64` — base64-encoded mobileprovision file
- `PLAY_SERVICE_ACCOUNT_JSON` — Google Play service account JSON
- `PROD_API_URL` — production API base URL
- `CODECOV_TOKEN` — Codecov upload token

## Codemagic

### codemagic.yaml

```yaml
# codemagic.yaml
workflows:
  flutter-prod-release:
    name: Flutter Production Release
    max_build_duration: 60
    instance_type: mac_mini_m2

    triggering:
      events:
        - push
      branch_patterns:
        - pattern: 'release/*'
          include: true
          source: true

    environment:
      flutter: 3.22.0
      java: 17
      xcode: latest
      cocoapods: default
      vars:
        BUNDLE_ID: com.example.app
        PACKAGE_NAME: com.example.app
      groups:
        - app_store_credentials    # APPLE_ID, APP_SPECIFIC_PASSWORD, TEAM_ID
        - google_play_credentials  # GCLOUD_SERVICE_ACCOUNT_CREDENTIALS
        - android_signing           # KEYSTORE, KEY_ALIAS, etc.
        - ios_signing               # Certificate + profile (Codemagic manages these)

    cache:
      cache_paths:
        - $FLUTTER_ROOT/bin/cache
        - $HOME/.pub-cache

    scripts:
      - name: Set up keystore
        script: |
          echo $KEYSTORE | base64 --decode > $CM_BUILD_DIR/android/app/keystore.jks
          cat >> "$CM_BUILD_DIR/android/key.properties" <<EOF
          storePassword=$KEYSTORE_PASSWORD
          keyPassword=$KEY_PASSWORD
          keyAlias=$KEY_ALIAS
          storeFile=keystore.jks
          EOF

      - name: Install dependencies
        script: flutter pub get

      - name: Analyze
        script: flutter analyze --fatal-infos

      - name: Run tests
        script: flutter test --coverage

      - name: Set build number
        script: |
          BUILD_NUMBER=$(($PROJECT_BUILD_NUMBER))
          echo "BUILD_NUMBER=$BUILD_NUMBER" >> $CM_ENV

      - name: Build Android AAB
        script: |
          flutter build appbundle \
            --flavor prod \
            --target lib/main_prod.dart \
            --release \
            --build-number=$BUILD_NUMBER \
            --dart-define=API_URL=$PROD_API_URL

      - name: Build iOS IPA
        script: |
          flutter build ipa \
            --flavor prod \
            --target lib/main_prod.dart \
            --release \
            --export-options-plist=ios/ExportOptions.plist \
            --build-number=$BUILD_NUMBER \
            --dart-define=API_URL=$PROD_API_URL

    artifacts:
      - build/app/outputs/bundle/prodRelease/*.aab
      - build/ios/ipa/*.ipa
      - flutter_drive.log
      - coverage/lcov.info

    publishing:
      google_play:
        credentials: $GCLOUD_SERVICE_ACCOUNT_CREDENTIALS
        track: internal
        submit_as_draft: true

      app_store_connect:
        api_key: $APP_STORE_CONNECT_PRIVATE_KEY
        key_id: $APP_STORE_CONNECT_KEY_IDENTIFIER
        issuer_id: $APP_STORE_CONNECT_ISSUER_ID
        submit_to_testflight: true
        beta_groups:
          - Internal Testers

      email:
        recipients:
          - team@example.com
        notify:
          success: true
          failure: true
```

## Caching Strategy

- Flutter SDK — key: `flutter-version` string — avoid re-downloading Flutter
- `~/.pub-cache` — key: hash of `pubspec.lock` — skip pub get on unchanged deps
- Gradle (`~/.gradle`) — key: hash of `*.gradle` files — speed up Android builds by ~2 min
- Xcode DerivedData — key: hash of `ios/Podfile.lock` — speed up iOS builds

**Cache invalidation:** always include a `restore-keys` fallback so a partial cache hit still provides benefit when a single dependency changes.

## Automated Testing in CI

```yaml
# Always run these before building
- flutter analyze --fatal-infos --fatal-warnings
- flutter test --coverage --reporter=github
# For integration tests (requires emulator/simulator):
- flutter test integration_test/ -d emulator-5554
```

**Test coverage gate:**
```yaml
- name: Check coverage threshold
  run: |
    COVERAGE=$(lcov --summary coverage/lcov.info 2>&1 | grep 'lines' | grep -oP '\d+\.\d+(?=%)')
    if (( $(echo "$COVERAGE < 70" | bc -l) )); then
      echo "Coverage $COVERAGE% is below threshold 70%"
      exit 1
    fi
```

## Version Bumping Automation

```yaml
# Auto-bump build number from CI run number
- name: Set version
  run: |
    # pubspec.yaml has: version: 1.2.3+0
    # Replace the build number part with CI run number
    sed -i "s/version: \(.*\)+.*/version: \1+${{ github.run_number }}/" pubspec.yaml

# Or use cider for semantic versioning
- name: Bump patch version
  run: |
    dart pub global activate cider
    cider bump patch
    cider bump build ${{ github.run_number }}
```

**Tagging releases:**
```yaml
- name: Tag release
  if: startsWith(github.ref, 'refs/heads/release/')
  run: |
    VERSION=$(grep 'version:' pubspec.yaml | sed 's/version: //' | cut -d'+' -f1)
    git tag "v$VERSION"
    git push origin "v$VERSION"
```

## Build Artifact Publishing

```yaml
# GitHub Releases
- name: Create GitHub Release
  uses: softprops/action-gh-release@v2
  if: startsWith(github.ref, 'refs/tags/v')
  with:
    files: |
      build/app/outputs/bundle/prodRelease/*.aab
      build/ios/ipa/*.ipa
    generate_release_notes: true

# Firebase App Distribution (staging)
- name: Upload to Firebase App Distribution
  uses: wzieba/Firebase-Distribution-Github-Action@v1
  with:
    appId: ${{ secrets.FIREBASE_APP_ID_ANDROID }}
    serviceCredentialsFileContent: ${{ secrets.FIREBASE_SERVICE_ACCOUNT }}
    groups: qa-team
    file: build/app/outputs/apk/staging/release/app-staging-release.apk
    releaseNotesFile: release_notes.txt
```
