# Flutter Crossplatform Development Plugin — Design Spec

## Overview

An opinionated Claude Code plugin for Flutter development, modeled after the `backend-development` plugin structure. Enforces strict architectural rules: BLoC-only state management, Clean Architecture with sealed classes, constructor injection, and explicit event transformers.

**Plugin name:** `flutter-crossplatform-development`
**Version:** 1.0.0
**License:** MIT

## Design Decisions

- **Strict and opinionated** — Same enforcement level as the project's local `flutter-developer` skill. No Cubit, no Hive, constructor injection only.
- **Latest stable Flutter/Dart** — No pinned version. Targets current stable, updated with plugin releases.
- **Persistence stack** — Drift for relational/type-safe SQL, SharedPreferences/SecureStorage for key-value. No Hive, no ObjectBox, no Isar.
- **Mirror backend-development structure** — 8 agents, 10 skills with reference files, 1 orchestration command.
- **Content seeded from local skill** — The existing `.claude/skills/flutter-developer/` provides the foundation for 5 skills (BLoC, View, Data, DI, Animation).

## Directory Structure

```
flutter-crossplatform-development/
├── .claude-plugin/
│   └── plugin.json
├── agents/
│   ├── flutter-architect.md
│   ├── flutter-state-expert.md
│   ├── flutter-ui-expert.md
│   ├── flutter-animation-expert.md
│   ├── flutter-testing-expert.md
│   ├── flutter-performance-engineer.md
│   ├── flutter-platform-engineer.md
│   └── flutter-data-engineer.md
├── skills/
│   ├── bloc-state-management/
│   │   ├── SKILL.md
│   │   └── references/
│   │       ├── bloc-core-rules.md
│   │       ├── event-design.md
│   │       ├── state-design.md
│   │       └── transformer-strategies.md
│   ├── architecture-patterns/
│   │   ├── SKILL.md
│   │   └── references/
│   │       ├── clean-architecture.md
│   │       ├── feature-driven-development.md
│   │       └── dependency-injection.md
│   ├── view-ui-layer/
│   │   ├── SKILL.md
│   │   └── references/
│   │       ├── widget-composition.md
│   │       ├── bloc-widgets.md
│   │       ├── responsive-adaptive.md
│   │       └── theming-design-system.md
│   ├── data-domain-layer/
│   │   ├── SKILL.md
│   │   └── references/
│   │       ├── entities-dtos.md
│   │       ├── repository-pattern.md
│   │       ├── drift-persistence.md
│   │       └── api-integration.md
│   ├── animation-performance/
│   │   ├── SKILL.md
│   │   └── references/
│   │       ├── animation-controllers.md
│   │       ├── custom-painter.md
│   │       ├── shaders-render-objects.md
│   │       └── performance-checklist.md
│   ├── testing-strategies/
│   │   ├── SKILL.md
│   │   └── references/
│   │       ├── unit-testing.md
│   │       ├── widget-testing.md
│   │       ├── golden-file-testing.md
│   │       └── integration-testing.md
│   ├── platform-integration/
│   │   ├── SKILL.md
│   │   └── references/
│   │       ├── platform-channels.md
│   │       ├── ffi-native-code.md
│   │       └── platform-specific-features.md
│   ├── dart-language-mastery/
│   │   ├── SKILL.md
│   │   └── references/
│   │       ├── patterns-records-sealed.md
│   │       ├── async-isolates.md
│   │       └── code-generation.md
│   ├── devops-deployment/
│   │   ├── SKILL.md
│   │   └── references/
│   │       ├── flavors-environments.md
│   │       ├── cicd-pipelines.md
│   │       └── app-store-deployment.md
│   └── security-compliance/
│       ├── SKILL.md
│       └── references/
│           ├── secure-storage.md
│           ├── network-security.md
│           └── compliance-gdpr.md
└── commands/
    └── flutter-feature.md
```

## Plugin Manifest

```json
{
  "name": "flutter-crossplatform-development",
  "version": "1.0.0",
  "description": "Opinionated Flutter development with BLoC state management, Clean Architecture, testing strategies, animation/performance optimization, and multi-platform deployment",
  "author": {
    "name": "Tamerlan Altynbek"
  },
  "license": "MIT"
}
```

## Global Rules (Enforced by All Agents and Skills)

1. **Never use Cubit** — BLoC only. Exceptions: stream-based state (WebSockets, location) or explicit learning phase.
2. **Constructor injection only** — Never use getIt or service locator inside BLoC. Repositories passed via constructor.
3. **Never expose DTOs to BLoC layer** — Always map to domain entities at repository boundary.
4. **Sealed classes + Equatable** — For all states and events. Immutable, no logic in states.
5. **Explicit transformers on every `on<>` handler** — Default is `sequential()`. No bare `on<Event>(handler)`.
6. **No Hive** — Use Drift for SQL, SharedPreferences/SecureStorage for key-value.
7. **Theme.of(context) always** — Never hardcode colors or text styles.
8. **Localization classes for all strings** — No hardcoded user-facing strings.
9. **StatelessWidget by default** — StatefulWidget only for ephemeral state (animations, text controllers).
10. **RepaintBoundary for animated content** — Never setState to drive CustomPainter.

## Agents

### flutter-architect
- **Model:** inherit
- **Trigger:** Architecture decisions, module boundaries, new feature structure, project setup
- **Expertise:** Clean Architecture layer separation, feature-driven development, package structure (`packages/domain/`, `packages/data/`), DI with CompositionRoot and AppScope, monorepo with Melos, dependency rule enforcement
- **Proactive use:** When creating new features, restructuring projects, or discussing module boundaries

### flutter-state-expert
- **Model:** inherit
- **Trigger:** BLoC/state management questions, race conditions, event flow, transformer selection
- **Expertise:** BLoC-only (no Cubit), sealed states/events with Equatable, transformer strategies (sequential/droppable/restartable), logically scoped event registration with switch expressions, BLoC-to-BLoC communication via repository streams, emit.forEach/onEach patterns
- **Proactive use:** When implementing business logic, discussing state shape, or debugging race conditions

### flutter-ui-expert
- **Model:** sonnet
- **Trigger:** Widget composition, screens, BlocBuilder/Listener/Consumer, navigation, theming, responsive design
- **Expertise:** BlocBuilder with switch expression on sealed states, BlocListener for side effects with private method extraction, BlocConsumer, buildWhen/listenWhen optimization, MultiBlocProvider/Listener, context extensions, Material 3 theming, Cupertino design, adaptive/responsive layouts with LayoutBuilder/MediaQuery
- **Proactive use:** When building screens, connecting UI to BLoC, or implementing design systems

### flutter-animation-expert
- **Model:** sonnet
- **Trigger:** Animations, CustomPainter, shaders, Rive/Lottie, Impeller, render objects
- **Expertise:** AnimationController with vsync, repaint parameter (never setState), RepaintBoundary isolation, fragment shaders (GPU), custom Ticker for precise frame control, RenderObject with markNeedsPaint, Paint reuse, batch drawing (drawPoints/drawAtlas), canvas save/restore, implicit vs explicit animations, CurvedAnimation, staggered animations
- **Proactive use:** When implementing animations or diagnosing rendering performance

### flutter-testing-expert
- **Model:** sonnet
- **Trigger:** Writing tests, TDD, widget testing, golden files, integration testing, coverage
- **Expertise:** BLoC unit tests with bloc_test, sealed state assertions, mockito for repositories, widget testing with testWidgets and sealed state matching, golden file setup and CI configuration, Patrol for E2E integration tests, test coverage analysis, TDD red-green-refactor for Flutter
- **Proactive use:** When implementing features (TDD), reviewing untested code, or setting up test infrastructure

### flutter-performance-engineer
- **Model:** sonnet
- **Trigger:** Performance issues, profiling, jank, memory leaks, bundle size, slow builds
- **Expertise:** Flutter DevTools profiling, widget rebuild minimization (const constructors, keys), Isolate.run for CPU work, Sliver lists for large datasets, tree shaking, Impeller migration strategies, memory profiling, frame rendering optimization (60/120fps), build optimization, app bundle size reduction
- **Proactive use:** When reviewing code for performance issues or diagnosing runtime problems

### flutter-platform-engineer
- **Model:** inherit
- **Trigger:** Platform channels, FFI, native plugins, iOS/Android/Web/Desktop specific features
- **Expertise:** MethodChannel, EventChannel, BasicMessageChannel with codec selection, Dart FFI for C/C++ with ffigen, iOS integration (Swift, Face ID, Haptics, Live Activities), Android integration (Kotlin, biometric, widgets, Material You), Web (PWA, JS interop), Desktop (system tray, menu bar, file system, window management), error handling across platform boundary
- **Proactive use:** When adding native features or building platform-specific functionality

### flutter-data-engineer
- **Model:** sonnet
- **Trigger:** Repositories, persistence, API integration, offline-first, caching, DTOs, mappers
- **Expertise:** Pure Dart entities with Equatable (no JSON, no Flutter), DTOs with json_serializable + snake_case, abstract repository interfaces with @useResult, @immutable implementations, mapper classes for complex/bidirectional conversions, Drift for type-safe SQL (setup, migrations, DAOs), SharedPreferences/SecureStorage for key-value, Dio + Retrofit for REST, domain exceptions at boundary (never expose DioException), offline-first sync, caching strategies
- **Proactive use:** When designing data flow, implementing repositories, or setting up persistence

## Skills

### 1. bloc-state-management
- **Name:** bloc-state-management
- **Description:** BLoC state management patterns with sealed classes, event transformers, and race condition prevention. Use when writing BLoC classes, events, states, transformers, or debugging state management issues. Trigger on: BLoC, Cubit, events, states, transformers, race conditions, state management, emit, sequential, droppable, restartable.
- **SKILL.md:** Architecture overview diagram (Widget ↔ BLoC ↔ Data), reference table with "read before answering" rule, quick reference of BLoC layer rules, key anti-patterns to flag immediately
- **References:**
  - `bloc-core-rules.md` — What BLoC is/isn't, layer separation, file structure (single vs multi-file), immutability rules, constructor injection, no public methods, no Flutter imports. **Seeded from local `bloc-rules.md`**
  - `event-design.md` — Sealed event classes, action-oriented naming, State Emitter mixins for reducing duplicate code, event grouping for shared transformers
  - `state-design.md` — Sealed states + Equatable, immutable final fields, props override pattern, shared base class properties (items, cursor, hasMore), state copy patterns, isProcessing getter, anti-patterns (mutable states, reference mutation, identical states)
  - `transformer-strategies.md` — sequential (default, most common), droppable (jump/login), restartable (search/autocomplete/streams), logically scoped registration with switch expressions, emit.forEach/onEach for repository streams, bloc_concurrency package

### 2. architecture-patterns
- **Name:** architecture-patterns
- **Description:** Flutter Clean Architecture, feature-driven development, and dependency injection patterns. Use when architecting Flutter apps, organizing features, setting up DI, or defining module boundaries. Trigger on: Clean Architecture, project structure, feature organization, DI, dependency injection, CompositionRoot, AppScope, packages, modules.
- **SKILL.md:** Layer diagram, reference table, quick rules per layer
- **References:**
  - `clean-architecture.md` — Layer separation (Presentation → Domain → Data), dependency rule (inner layers don't know outer), package structure (`packages/domain/[feature]/`, `packages/data/[feature]/`), what goes where
  - `feature-driven-development.md` — Feature-first directory structure, feature isolation, shared kernel for cross-feature code, monorepo management with Melos, feature container pattern
  - `dependency-injection.md` — Pure DI (no service locator), CompositionRoot pattern, Dependencies container class, CoreDependenciesBuilder/FeatureDependenciesBuilder, AppRunner initialization, AppScope with MultiProvider, context.di extensions, feature containers for logical grouping. **Seeded from local `di-reference.md`**

### 3. view-ui-layer
- **Name:** view-ui-layer
- **Description:** Flutter View/UI layer patterns with BlocBuilder, BlocListener, widget composition, and responsive design. Use when writing screens, widgets, connecting UI to BLoC, theming, or building responsive layouts. Trigger on: screens, widgets, BlocBuilder, BlocListener, BlocConsumer, navigation, theming, responsive, adaptive, Material 3, Cupertino.
- **SKILL.md:** View layer responsibilities, directory structure, reference table, quick rules
- **References:**
  - `widget-composition.md` — Small focused widgets, extract private widget classes, StatelessWidget by default, StatefulWidget only for ephemeral state, const constructors, key usage strategy
  - `bloc-widgets.md` — BlocBuilder with switch expression on sealed states, BlocListener for side effects with private method extraction, BlocConsumer, buildWhen/listenWhen for rebuild optimization, MultiBlocProvider/MultiBlocListener, context extensions for boilerplate reduction. **Seeded from local `view-rules.md`**
  - `responsive-adaptive.md` — LayoutBuilder, MediaQuery, breakpoint system, adaptive widgets (platform-aware), responsive design for mobile/tablet/desktop, orientation handling
  - `theming-design-system.md` — Theme.of(context) always, Material 3 color scheme and typography, dark/light theme setup, custom ThemeExtension for app-specific tokens, Cupertino theming, design system implementation

### 4. data-domain-layer
- **Name:** data-domain-layer
- **Description:** Data and Domain layer patterns with entities, DTOs, repositories, API integration, and Drift persistence. Use when working with data models, repositories, API clients, local storage, mappers, or error handling. Trigger on: entities, DTOs, repositories, API clients, Drift, Dio, Retrofit, mappers, JSON serialization, offline, caching.
- **SKILL.md:** Layer overview (Domain vs Data), directory structure, reference table, quick rules
- **References:**
  - `entities-dtos.md` — Pure Dart entities with Equatable (no JSON, no Flutter imports), DTOs with json_serializable + FieldRename.snake, strict separation, copyWith patterns. **Seeded from local `data-rules.md`**
  - `repository-pattern.md` — Abstract interfaces with @useResult, @immutable implementations, constructor-injected data sources, map DTOs to entities before returning, domain exceptions at boundary, never expose DioException, caching strategy in repository
  - `drift-persistence.md` — Drift setup and configuration, type-safe table definitions, DAOs for organized queries, schema migrations, reactive queries with watch(), SharedPreferences for simple key-value, flutter_secure_storage for sensitive data
  - `api-integration.md` — Dio configuration with BaseOptions, Retrofit abstract client with code generation, interceptors (auth, logging, retry), error mapping (DioException → domain exceptions), offline-first sync patterns, request cancellation

### 5. animation-performance
- **Name:** animation-performance
- **Description:** Flutter animation and rendering performance with AnimationController, CustomPainter, shaders, and RepaintBoundary. Use when implementing animations, CustomPainter, diagnosing jank, or optimizing rendering. Trigger on: AnimationController, CustomPainter, Tween, RepaintBoundary, shaders, RenderObject, frame rates, performance, animation, Impeller, jank.
- **SKILL.md:** Performance rules table, common pitfalls to flag, reference table
- **References:**
  - `animation-controllers.md` — Implicit animations (AnimatedContainer, AnimatedOpacity) for simple cases, explicit animations (AnimationController + CurvedAnimation + Tween), staggered animations, Hero animations, Rive/Lottie integration. **Seeded from local `animation-rules.md`**
  - `custom-painter.md` — repaint parameter (never setState to drive painter), Paint object reuse (define as field, never create in paint()), batch drawing (drawPoints/drawAtlas for single GPU call), canvas save/restore for layer management
  - `shaders-render-objects.md` — Fragment shaders via FragmentProgram.fromAsset (GPU-accelerated), custom Ticker for precise frame control, RenderObject with markNeedsPaint (bypass widget tree), LeafRenderObjectWidget pattern, animation listener wiring
  - `performance-checklist.md` — RepaintBoundary isolation for animated content, never change tree structure during animation (use Transform/Opacity), AnimatedBuilder over setState, let Flutter manage refresh rate (no Timer.periodic), performance monitoring with Timeline, DevTools profiling workflow

### 6. testing-strategies
- **Name:** testing-strategies
- **Description:** Flutter testing with unit tests, widget tests, golden files, and integration tests. Use when writing tests, setting up TDD, debugging test failures, or configuring test infrastructure. Trigger on: tests, TDD, unit testing, widget testing, golden files, integration testing, Patrol, bloc_test, mockito, coverage.
- **SKILL.md:** Testing pyramid for Flutter, reference table, quick rules per test type
- **References:**
  - `unit-testing.md` — BLoC testing with bloc_test package (act/expect on state sequences), testing transformer behavior (sequential/droppable/restartable), mockito for repository mocking, testing domain exceptions, testing mappers
  - `widget-testing.md` — testWidgets setup, pumpWidget with BlocProvider, finding widgets (find.byType, find.text), tapping and entering text, verifying BlocBuilder renders correct widget per sealed state, pump vs pumpAndSettle
  - `golden-file-testing.md` — Golden file setup, matchesGoldenFile, font loading for deterministic renders, updating goldens (--update-goldens), CI configuration for golden tests, platform-specific golden management
  - `integration-testing.md` — Patrol setup and configuration, native interaction testing, integration_test package basics, test drivers, running on device farms, CI integration for integration tests

### 7. platform-integration
- **Name:** platform-integration
- **Description:** Flutter platform channels, FFI, and platform-specific feature implementation. Use when integrating native code, building platform channels, using FFI, or implementing iOS/Android/Web/Desktop specific features. Trigger on: platform channels, MethodChannel, EventChannel, FFI, native code, Swift, Kotlin, JS interop, desktop, web platform.
- **SKILL.md:** Platform abstraction overview, reference table, when to use which channel type
- **References:**
  - `platform-channels.md` — MethodChannel for request/response, EventChannel for streams from native, BasicMessageChannel for simple messaging, codec selection (StandardMessageCodec, JSONMessageCodec), error handling across boundary, platform-specific implementations (Swift/Kotlin)
  - `ffi-native-code.md` — Dart FFI for C/C++ libraries, ffigen for automatic binding generation, memory management (malloc/free), struct passing and callbacks, NativeCallable for asynchronous callbacks, package:ffi utilities
  - `platform-specific-features.md` — iOS: Face ID/Touch ID, Haptic feedback, Live Activities, App Clips, WidgetKit. Android: BiometricPrompt, Home screen widgets, Material You dynamic theming, App Links. Web: PWA manifest, JS interop with dart:js_interop, web-specific rendering. Desktop: System tray, menu bar, window management, file system access, keyboard shortcuts

### 8. dart-language-mastery
- **Name:** dart-language-mastery
- **Description:** Advanced Dart language features including patterns, records, sealed classes, async programming, and code generation. Use when working with Dart 3 features, async patterns, Isolates, or build_runner code generation. Trigger on: Dart 3, patterns, records, sealed classes, switch expressions, async, Future, Stream, Isolate, code generation, build_runner, freezed.
- **SKILL.md:** Dart feature overview, reference table, quick examples of key patterns
- **References:**
  - `patterns-records-sealed.md` — Pattern matching with switch expressions (exhaustiveness), destructuring (positional and named), records for multiple return values, sealed class hierarchies for type-safe unions, guard clauses in patterns, logical/relational patterns
  - `async-isolates.md` — Future and async/await best practices, Stream and StreamController, Isolate.run() for CPU-intensive work, compute() shorthand, Zone error handling, stream transformers, completer usage
  - `code-generation.md` — build_runner setup and workflow, json_serializable for DTOs, freezed for immutable data classes with union types, auto_route for type-safe navigation, injectable for DI code generation, custom Builder implementation

### 9. devops-deployment
- **Name:** devops-deployment
- **Description:** Flutter CI/CD, flavors, code signing, and app store deployment. Use when setting up build pipelines, configuring flavors, managing code signing, or deploying to app stores. Trigger on: CI/CD, GitHub Actions, Codemagic, flavors, code signing, certificates, TestFlight, Play Store, app store, deployment, Firebase distribution.
- **SKILL.md:** Deployment overview, reference table, flavor/environment quick reference
- **References:**
  - `flavors-environments.md` — Flutter flavors (dev/staging/prod), --dart-define and --dart-define-from-file, per-flavor bundle IDs and icons, Android productFlavors, iOS schemes/configurations, environment-specific API endpoints
  - `cicd-pipelines.md` — GitHub Actions workflow for Flutter (test → build → deploy), Codemagic configuration, Bitrise workflows, caching dependencies, automated testing in CI, build artifact publishing, version bumping automation
  - `app-store-deployment.md` — iOS: provisioning profiles, certificates, TestFlight distribution, App Store Connect submission. Google Play: upload key signing, Play Console tracks (internal/alpha/beta/production). Web: hosting options (Firebase Hosting, Vercel, Cloudflare). Desktop: DMG/MSI/AppImage packaging, code signing, notarization (macOS)

### 10. security-compliance
- **Name:** security-compliance
- **Description:** Flutter security best practices including secure storage, network security, and compliance. Use when implementing authentication, securing data, hardening the app, or ensuring regulatory compliance. Trigger on: secure storage, Keychain, KeyStore, certificate pinning, biometric auth, GDPR, obfuscation, encryption, API security, tokens.
- **SKILL.md:** Security checklist, reference table, critical rules
- **References:**
  - `secure-storage.md` — flutter_secure_storage setup, iOS Keychain integration, Android KeyStore/EncryptedSharedPreferences, token lifecycle management (access/refresh), encryption at rest, secure deletion
  - `network-security.md` — Certificate pinning with Dio interceptor, HTTPS enforcement, API key management (never in source), request signing for API integrity, network security config (Android), App Transport Security (iOS)
  - `compliance-gdpr.md` — GDPR data handling principles, privacy-by-design, user consent management, right to deletion implementation, data minimization, --obfuscate and --split-debug-info for release builds, audit logging

## Orchestration Command: flutter-feature

### Frontmatter
```yaml
---
description: "Orchestrate end-to-end Flutter feature development from requirements to deployment"
argument-hint: "<feature description> [--architecture clean|feature-driven] [--complexity simple|medium|complex]"
---
```

### Pre-flight Checks
1. Check for `.flutter-dev/state.json` — resume or start fresh
2. Initialize state with feature name, architecture choice, complexity
3. Parse feature description from arguments

### Phase 1: Discovery (Steps 1-3) — Sequential

**Step 1: Requirements & Architecture**
- Agent: flutter-architect
- Output: `.flutter-dev/01-architecture.md`
- Purpose: Feature scope, layer boundaries, package structure, DI plan

**Step 2: State Management Design**
- Agent: flutter-state-expert
- Input: Step 1 output
- Output: `.flutter-dev/02-state-design.md`
- Purpose: BLoC definitions (events, states, transformers), repository interfaces needed

**Step 3: Data Layer Design**
- Agent: flutter-data-engineer
- Input: Steps 1-2 output
- Output: `.flutter-dev/03-data-design.md`
- Purpose: Entities, DTOs, repository implementations, API contracts, Drift schemas if needed

### CHECKPOINT 1
User reviews architecture, state design, and data layer. Options: approve, request changes, pause.

### Phase 2: Implementation (Steps 4-6) — Parallel where possible

**Step 4a: BLoC Implementation** (parallel with 4b)
- Agent: flutter-state-expert
- Input: Steps 1-3 output
- Output: `.flutter-dev/04a-bloc.md`

**Step 4b: Data Layer Implementation** (parallel with 4a)
- Agent: flutter-data-engineer
- Input: Steps 1-3 output
- Output: `.flutter-dev/04b-data.md`

**Step 4c: UI Implementation** (depends on 4a + 4b)
- Agent: flutter-ui-expert
- Input: Steps 1-3 + 4a + 4b output
- Output: `.flutter-dev/04c-ui.md`

**Step 5: Testing** (depends on 4a-4c)
- Agent: flutter-testing-expert
- Input: All prior outputs
- Output: `.flutter-dev/05-tests.md`

**Step 6: Performance Review** (depends on 4c)
- Agent: flutter-performance-engineer
- Input: Steps 4a-4c output
- Output: `.flutter-dev/06-performance.md`

### CHECKPOINT 2
User reviews implementation, tests, and performance findings. Options: approve, request changes, pause.

### Phase 3: Delivery (Step 7) — Sequential

**Step 7: Platform & Deployment**
- Agent: flutter-platform-engineer
- Input: All prior outputs
- Output: `.flutter-dev/07-platform.md`
- Purpose: Platform-specific adjustments, flavor config, CI/CD setup, store readiness

### State Management
`.flutter-dev/state.json` schema:
```json
{
  "feature": "<feature description>",
  "status": "in_progress",
  "architecture": "clean",
  "complexity": "medium",
  "current_step": 1,
  "current_phase": 1,
  "completed_steps": [],
  "files_created": [],
  "started_at": "ISO_TIMESTAMP",
  "last_updated": "ISO_TIMESTAMP"
}
```

### Behavioral Rules
1. Execute steps in order — no skipping, reordering, or merging
2. Write output files — each step produces its file before proceeding
3. Stop at checkpoints — wait for explicit user approval
4. Halt on failure — present error, ask how to proceed
5. Use only plugin-local agents — no cross-plugin dependencies
6. Never enter plan mode autonomously

### Completion Summary
Lists all files created, implementation summary by phase, success criteria:
- Architecture reviewed and approved before implementation
- BLoC classes use explicit transformers, sealed states/events
- All DTOs mapped to entities at repository boundary
- Widget tests cover all sealed state branches
- Performance review completed with recommendations
- Platform-specific adjustments documented

## Content Seeding

The following local skill files seed the initial plugin content:

| Local file | Seeds plugin reference |
|---|---|
| `.claude/skills/flutter-developer/references/bloc-rules.md` | `bloc-state-management/references/bloc-core-rules.md` |
| `.claude/skills/flutter-developer/references/view-rules.md` | `view-ui-layer/references/bloc-widgets.md` |
| `.claude/skills/flutter-developer/references/data-rules.md` | `data-domain-layer/references/entities-dtos.md` + `repository-pattern.md` |
| `.claude/skills/flutter-developer/references/di-reference.md` | `architecture-patterns/references/dependency-injection.md` |
| `.claude/skills/flutter-developer/references/animation-rules.md` | `animation-performance/references/animation-controllers.md` + `custom-painter.md` |

Seeded content will be expanded with additional examples, edge cases, and cross-references to other plugin skills.

## File Count Summary

| Component | Count |
|---|---|
| Agents | 8 |
| Skills | 10 |
| Reference files | 35 |
| Commands | 1 |
| **Total files** | **55** (including SKILL.md files and plugin.json) |
