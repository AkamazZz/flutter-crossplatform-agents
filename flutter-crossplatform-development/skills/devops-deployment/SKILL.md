---
name: devops-deployment
description: >
  Flutter CI/CD, flavors, code signing, and app store deployment. Use when setting up build
  pipelines, configuring flavors, managing code signing, or deploying to app stores. Trigger on:
  CI/CD, GitHub Actions, Codemagic, flavors, code signing, certificates, TestFlight, Play Store,
  app store, deployment, Firebase distribution.
---

# DevOps & Deployment

Flutter deployment follows a structured pipeline: **build → test → deploy**. Each stage has platform-specific requirements, and a well-configured pipeline catches issues early, automates signing, and delivers to end users reliably across iOS, Android, web, and desktop.

## Pipeline Overview

```
Source Code
    │
    ▼
[Build] ──── Flavors (dev/staging/prod)
    │         Code signing (iOS certs, Android keystore)
    │         --dart-define environment injection
    │
    ▼
[Test] ───── Unit tests, widget tests, integration tests
    │         Coverage reports
    │         Static analysis (flutter analyze)
    │
    ▼
[Deploy] ─── iOS: TestFlight → App Store
              Android: Internal → Alpha → Beta → Production
              Web: Firebase Hosting / Vercel / Cloudflare
              Desktop: DMG / MSI / AppImage
```

## Reference Documents

| Document | Contents | When to Use |
|---|---|---|
| [flavors-environments.md](references/flavors-environments.md) | Flutter flavors, --dart-define, per-flavor bundle IDs/icons, Android productFlavors, iOS schemes | Setting up dev/staging/prod environments |
| [cicd-pipelines.md](references/cicd-pipelines.md) | GitHub Actions YAML, Codemagic config, caching, versioning, branch strategy | Configuring automated build and deploy pipelines |
| [app-store-deployment.md](references/app-store-deployment.md) | iOS provisioning/certificates/TestFlight, Google Play tracks, web hosting, desktop packaging | Submitting to app stores and hosting platforms |

## Quick Reference

**Run a specific flavor:**
```bash
flutter run --flavor dev --target lib/main_dev.dart
flutter build apk --flavor prod --target lib/main_prod.dart
flutter build ipa --flavor prod --target lib/main_prod.dart
```

**Inject environment variables:**
```bash
flutter build apk \
  --dart-define=API_URL=https://api.example.com \
  --dart-define=ENVIRONMENT=production
```

**Read dart-defines in code:**
```dart
const apiUrl = String.fromEnvironment('API_URL', defaultValue: 'https://dev.api.example.com');
const environment = String.fromEnvironment('ENVIRONMENT', defaultValue: 'dev');
```

**Version bump before build:**
```bash
# Set version in pubspec.yaml: version: 1.2.3+45
flutter build apk --build-name=1.2.3 --build-number=45
```

**Common CI commands:**
```bash
flutter pub get
flutter analyze
flutter test --coverage
flutter build apk --release
```

## Anti-Patterns

- **Hardcoding environment URLs** — always use `--dart-define` or `--dart-define-from-file`; never commit `.env` files with secrets
- **Committing signing credentials** — store keystores and certificates in CI secrets or a secrets manager, never in source control
- **Single-track releases** — always use staged rollouts (internal → alpha → beta → production) to catch issues before full release
- **Skipping tests in CI** — a pipeline that only builds is not a pipeline; unit + widget tests must run on every PR
- **Manual version bumping** — automate build numbers from CI run numbers or git tags to avoid human error
- **Using Hive for build configuration** — do not add Hive as a dependency for storing flavor/environment config; use dart-defines and compile-time constants instead

