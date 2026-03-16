# Flutter Crossplatform Development Plugin — Implementation Plan

> **For agentic workers:** REQUIRED: Use superpowers:subagent-driven-development (if subagents available) or superpowers:executing-plans to implement this plan. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build an opinionated Claude Code plugin for Flutter development with 8 agents, 10 skills (36 references), and 1 orchestration command.

**Architecture:** All files are markdown content (no compilable code). Plugin uses directory-convention discovery. Content for 5 skills is seeded from the existing local `.claude/skills/flutter-developer/` skill and expanded. Remaining 5 skills and all agents are written from scratch using the spec as the source of truth.

**Tech Stack:** Claude Code plugin system (markdown + YAML frontmatter), modeled after `backend-development` plugin at `~/.claude/plugins/cache/claude-plugins-official/backend-development/`

**Spec:** `docs/superpowers/specs/2026-03-16-flutter-crossplatform-development-plugin-design.md`

**Reference plugin (read for format examples):** `~/.claude/plugins/cache/claude-plugins-official/backend-development/1.3.1/`

**Local skill to seed from:** `.claude/skills/flutter-developer/` (files: `SKILL.md`, `references/bloc-rules.md`, `references/view-rules.md`, `references/data-rules.md`, `references/di-reference.md`, `references/animation-rules.md`)

---

## Important Notes for Implementers

1. **This is a content plugin, not a code project.** There are no tests to run, no builds to execute. Quality is verified by structural compliance with the spec's file format requirements.
2. **Read the reference plugin** before writing any file. The backend-development plugin at the path above shows exact formatting, line counts, and depth expected for agents (~150-310 lines), skills (~50-110 lines for SKILL.md), and commands (~500 lines).
3. **Global Rules must appear in every agent and skill.** The 10 global rules from the spec (lines 123-132) must be embedded in every agent's Behavioral Traits and referenced in every skill's Quick Reference where relevant. Use this mapping:
   - **bloc-state-management**: Rules 1 (no Cubit), 2 (constructor injection), 4 (sealed+Equatable), 5 (explicit transformers)
   - **architecture-patterns**: Rules 2 (constructor injection), 3 (no DTO exposure)
   - **view-ui-layer**: Rules 7 (Theme.of), 8 (localization), 9 (StatelessWidget default)
   - **data-domain-layer**: Rules 3 (no DTO exposure), 6 (no Hive)
   - **animation-performance**: Rule 10 (RepaintBoundary, no setState for painter)
   - **testing-strategies**: Rules 1, 4, 5 (test that BLoCs follow these rules)
   - **platform-integration**: Rule 2 (constructor injection for native abstractions)
   - **dart-language-mastery**: Rules 4 (sealed classes), 5 (transformers)
   - **devops-deployment**: Rule 6 (no Hive in dependencies)
   - **security-compliance**: Rule 6 (SecureStorage, not SharedPreferences for sensitive data)
4. **Seeded content must be expanded**, not just copied. The local skill files are a starting point. Add more examples, edge cases, and cross-references to other plugin skills.
5. **All paths below are relative to the plugin root:** `flutter-crossplatform-development/`

---

## Chunk 1: Foundation + Plugin Scaffold

### Task 1: Create plugin directory structure and manifest

**Files:**
- Create: `flutter-crossplatform-development/.claude-plugin/plugin.json`

- [ ] **Step 1: Create all directories**

```bash
mkdir -p flutter-crossplatform-development/.claude-plugin
mkdir -p flutter-crossplatform-development/agents
mkdir -p flutter-crossplatform-development/commands
mkdir -p flutter-crossplatform-development/skills/bloc-state-management/references
mkdir -p flutter-crossplatform-development/skills/architecture-patterns/references
mkdir -p flutter-crossplatform-development/skills/view-ui-layer/references
mkdir -p flutter-crossplatform-development/skills/data-domain-layer/references
mkdir -p flutter-crossplatform-development/skills/animation-performance/references
mkdir -p flutter-crossplatform-development/skills/testing-strategies/references
mkdir -p flutter-crossplatform-development/skills/platform-integration/references
mkdir -p flutter-crossplatform-development/skills/dart-language-mastery/references
mkdir -p flutter-crossplatform-development/skills/devops-deployment/references
mkdir -p flutter-crossplatform-development/skills/security-compliance/references
```

- [ ] **Step 2: Write plugin.json**

Create `flutter-crossplatform-development/.claude-plugin/plugin.json`:

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

- [ ] **Step 3: Commit scaffold**

```bash
git add flutter-crossplatform-development/
git commit -m "scaffold: create flutter-crossplatform-development plugin directory structure"
```

---

## Chunk 2: Seeded Skills (5 skills from local flutter-developer)

These skills are seeded from existing local files. Read the local source, then expand with additional content per the spec.

### Task 2: bloc-state-management skill (seeded)

**Files:**
- Create: `skills/bloc-state-management/SKILL.md`
- Create: `skills/bloc-state-management/references/bloc-core-rules.md`
- Create: `skills/bloc-state-management/references/event-design.md`
- Create: `skills/bloc-state-management/references/state-design.md`
- Create: `skills/bloc-state-management/references/transformer-strategies.md`

**Source to read first:**
- `.claude/skills/flutter-developer/SKILL.md` — Use as template for SKILL.md structure (reference table pattern)
- `.claude/skills/flutter-developer/references/bloc-rules.md` — Seeds `bloc-core-rules.md`

- [ ] **Step 1: Write SKILL.md**

Create `skills/bloc-state-management/SKILL.md` with:
- Frontmatter: `name: bloc-state-management`, verbose `description` with all trigger keywords from spec line 239
- Body following the SKILL.md Body Structure from spec lines 224-228:
  1. Title: `# BLoC State Management`
  2. Architecture overview: Widget ↔ BLoC ↔ Data layer diagram (copy from local SKILL.md lines 22-26)
  3. Reference table: 4-row table mapping topics → reference files → "When to read" (same format as local SKILL.md lines 30-38)
  4. Quick Reference table: BLoC layer rules from spec (State shape, Event shape, Transformers, Repositories, BLoC↔BLoC, Cubit, Public methods, Flutter imports)
  5. Key Anti-Patterns: 6 items from local SKILL.md lines 57-63

- [ ] **Step 2: Write bloc-core-rules.md**

Create `skills/bloc-state-management/references/bloc-core-rules.md`:
- Copy local `bloc-rules.md` content (sections 1-8)
- This is the largest reference file — keep all content, expand with:
  - Cross-reference to `transformer-strategies.md` for detailed transformer docs
  - Cross-reference to `event-design.md` and `state-design.md` for detailed patterns
  - Note about Global Rule 1 (never Cubit) and Global Rule 2 (constructor injection)

- [ ] **Step 3: Write event-design.md**

Create `skills/bloc-state-management/references/event-design.md`:
- Extract event-related content from local `bloc-rules.md` section 5
- Expand with:
  - Full sealed event class example with multiple event types
  - Action-oriented naming conventions (Load, Refresh, Delete, Submit — not Loaded, Refreshed)
  - State Emitter mixin pattern for reducing duplicate emit code
  - Event grouping pattern for shared transformers (switch expression in `on<>`)
  - Anti-patterns: events carrying too much data, events named after state changes

- [ ] **Step 4: Write state-design.md**

Create `skills/bloc-state-management/references/state-design.md`:
- Extract state-related content from local `bloc-rules.md` section 4
- Expand with:
  - Full sealed state hierarchy example with base class shared properties
  - props override pattern with super.props
  - isProcessing / hasError convenience getters
  - State copy pattern (always copy unchanged data between states)
  - Anti-patterns with code examples: mutable states, reference mutation, identical states, too many states

- [ ] **Step 5: Write transformer-strategies.md**

Create `skills/bloc-state-management/references/transformer-strategies.md`:
- Extract transformer content from local `bloc-rules.md` section 6
- Expand with:
  - bloc_concurrency package import and setup
  - sequential() — detailed example (auth flow, data updates)
  - droppable() — detailed example (jump action, login button)
  - restartable() — detailed example (search input, autocomplete, stream subscriptions with emit.forEach)
  - Logically scoped registration: full example showing multiple `on<>` with switch expressions grouping related events
  - emit.forEach and emit.onEach patterns for repository stream subscriptions
  - Decision flowchart: "which transformer do I need?"

- [ ] **Step 6: Commit**

```bash
git add skills/bloc-state-management/
git commit -m "feat: add bloc-state-management skill (seeded from local flutter-developer)"
```

### Task 3: view-ui-layer skill (seeded)

**Files:**
- Create: `skills/view-ui-layer/SKILL.md`
- Create: `skills/view-ui-layer/references/widget-composition.md`
- Create: `skills/view-ui-layer/references/bloc-widgets.md`
- Create: `skills/view-ui-layer/references/responsive-adaptive.md`
- Create: `skills/view-ui-layer/references/navigation.md`
- Create: `skills/view-ui-layer/references/theming-design-system.md`

**Source to read first:**
- `.claude/skills/flutter-developer/references/view-rules.md` — Seeds `bloc-widgets.md`

- [ ] **Step 1: Write SKILL.md**

Create `skills/view-ui-layer/SKILL.md` with:
- Frontmatter: `name: view-ui-layer`, verbose description with all trigger keywords from spec line 258
- Body:
  1. Title: `# View / UI Layer`
  2. Architecture overview: View layer responsibilities (rendering UI, handling interactions, dispatching events)
  3. Directory structure: `view/screens/` and `view/widgets/` pattern from local view-rules.md lines 12-17
  4. Reference table: 5-row table for the 5 reference files
  5. Quick Reference: rules from spec (Widget size, Statefulness, State rendering, Side effects, Boilerplate, Rebuild optimisation, Styling, Strings)
  6. Key Anti-Patterns: hardcoded colors, business logic in widgets, BlocBuilder without exhaustive switch, setState for BLoC state

- [ ] **Step 2: Write widget-composition.md**

New content (not seeded). Cover:
- Small focused widgets principle with before/after example (large build method → extracted private widget classes)
- StatelessWidget by default rule with examples of when StatefulWidget is acceptable
- const constructor usage for optimal rebuild performance
- Key usage strategy: when to use ValueKey, ObjectKey, UniqueKey
- Private widget class naming convention (`_FeatureHeader`, `_FeatureListItem`)

- [ ] **Step 3: Write bloc-widgets.md**

Seed from local `view-rules.md`, expand with:
- Full BlocBuilder + switch expression example (from local lines 37-99)
- BlocListener with pattern matching and method extraction (from local lines 106-141)
- BlocConsumer example (from local lines 148-163)
- Context extensions pattern (from local lines 169-187)
- buildWhen / listenWhen optimization (from local lines 193-229)
- MultiBlocProvider / MultiBlocListener (from local lines 233-269)
- Best practices section (from local lines 273-318)
- Add cross-reference to `bloc-state-management` skill for state/event design

- [ ] **Step 4: Write responsive-adaptive.md**

New content. Cover:
- LayoutBuilder for parent-constraint-based layout
- MediaQuery for screen-level breakpoints
- Breakpoint system: mobile (<600), tablet (600-1200), desktop (>1200)
- Adaptive widgets: platform-aware switches (iOS Cupertino vs Android Material)
- OrientationBuilder for landscape/portrait
- Responsive design patterns: single-column → multi-column, bottom nav → side nav
- SafeArea usage

- [ ] **Step 5: Write navigation.md**

New content. Cover:
- go_router setup: GoRouter configuration, route tree
- Route types: GoRoute, ShellRoute for nested navigation
- Type-safe paths with path parameters and query parameters
- Deep linking configuration (iOS universal links, Android app links)
- Redirect guards for authentication (redirect to login if unauthenticated)
- Navigator 2.0 fundamentals (Router, RouteInformationParser, RouterDelegate)
- Navigation with BLoC: listening to auth state for redirects
- Anti-pattern: using Navigator.push directly instead of go_router

- [ ] **Step 6: Write theming-design-system.md**

New content. Cover:
- Theme.of(context) always — never hardcode colors/text styles (Global Rule 7)
- Material 3 ColorScheme setup (seed color, dynamic theming)
- Typography with TextTheme
- Dark/light theme configuration in MaterialApp
- Custom ThemeExtension for app-specific design tokens
- Cupertino theming for iOS-specific screens
- Design system implementation: spacing, radius, elevation constants via ThemeExtension

- [ ] **Step 7: Commit**

```bash
git add skills/view-ui-layer/
git commit -m "feat: add view-ui-layer skill with navigation and theming (seeded from local flutter-developer)"
```

### Task 4: data-domain-layer skill (seeded, split)

**Files:**
- Create: `skills/data-domain-layer/SKILL.md`
- Create: `skills/data-domain-layer/references/entities-dtos.md`
- Create: `skills/data-domain-layer/references/repository-pattern.md`
- Create: `skills/data-domain-layer/references/drift-persistence.md`
- Create: `skills/data-domain-layer/references/api-integration.md`

**Source to read first:**
- `.claude/skills/flutter-developer/references/data-rules.md` — Seeds `entities-dtos.md` (sections 1-5) and `repository-pattern.md` (sections 6-11)

**Split boundary (from spec lines 460-463):**
- Sections 1-5 (Layer Overview, Directory Structure, Entities, Repository Interfaces, DTOs) → `entities-dtos.md`
- Sections 6-11 (Mappers, Data Sources, Repository Implementation, Error Handling, DI, Best Practices) → `repository-pattern.md`

- [ ] **Step 1: Write SKILL.md**

Create `skills/data-domain-layer/SKILL.md` with:
- Frontmatter: `name: data-domain-layer`, verbose description from spec line 269
- Body:
  1. Title: `# Data & Domain Layers`
  2. Architecture overview: Domain (pure business logic) vs Data (infrastructure) from local data-rules.md section 1
  3. Directory structure from local data-rules.md section 2
  4. Reference table: 4 rows for reference files
  5. Quick Reference: rules from spec (Entities, DTOs, Repository interface, Repository impl, Mappers, Exceptions, BLoC exposure)
  6. Anti-Patterns: expose DTOs to BLoC, put business logic in repos/DTOs, import domain into data sources

- [ ] **Step 2: Write entities-dtos.md**

Seed from local `data-rules.md` sections 1-5. Include:
- Layer overview and directory structure
- Entity rules: pure Dart, Equatable, no JSON, no Flutter imports, copyWith, business validation
- Full entity example (from local lines 43-69)
- Repository interface rules: abstract, @useResult, returns entities/primitives
- Full interface example (from local lines 80-93)
- DTO rules: json_serializable, FieldRename.snake, no business logic
- Full DTO example (from local lines 105-127)
- Add: clear statement that entities and DTOs must NEVER be the same class

- [ ] **Step 3: Write repository-pattern.md**

Seed from local `data-rules.md` sections 6-11. Include:
- Mapper classes: when to use, bidirectional mapping, complex mappers with dependencies (from local lines 139-183)
- Data sources: API client with Retrofit (from local lines 195-213), local storage (from local lines 217-238)
- Repository implementation: @immutable, inject sources, map at boundary, cache strategy (from local lines 250-316)
- Error handling: DomainException hierarchy, exception mapping from DioException (from local lines 319-351)
- DI setup (from local lines 355-371)
- Best practices checklist (from local lines 374-386)
- Add cross-reference to `architecture-patterns/dependency-injection` for DI details

- [ ] **Step 4: Write drift-persistence.md**

New content. Cover:
- Drift setup: pubspec dependency, database class definition
- Table definitions with type-safe columns
- DAOs for organized query access
- Schema migrations (stepByStep, sequential)
- Reactive queries with watch() returning Stream
- Transaction support
- SharedPreferences for simple key-value (non-sensitive)
- flutter_secure_storage for tokens/credentials
- When to use Drift vs SharedPreferences decision guide
- No Hive note (Global Rule 6)

- [ ] **Step 5: Write api-integration.md**

New content. Cover:
- Dio setup with BaseOptions (baseUrl, timeouts, headers)
- Retrofit abstract client with @RestApi, code generation setup
- Interceptors: auth token injection, request/response logging, retry logic
- Error mapping: DioException → domain exceptions (status code switch)
- Request cancellation with CancelToken
- Offline-first sync patterns: cache-then-network, queue failed requests
- Multipart upload handling
- Anti-pattern: catching DioException in BLoC (must be caught in repository)

- [ ] **Step 6: Commit**

```bash
git add skills/data-domain-layer/
git commit -m "feat: add data-domain-layer skill (seeded from local flutter-developer, split per spec)"
```

### Task 5: architecture-patterns skill (seeded)

**Files:**
- Create: `skills/architecture-patterns/SKILL.md`
- Create: `skills/architecture-patterns/references/clean-architecture.md`
- Create: `skills/architecture-patterns/references/feature-driven-development.md`
- Create: `skills/architecture-patterns/references/dependency-injection.md`

**Source to read first:**
- `.claude/skills/flutter-developer/references/di-reference.md` — Seeds `dependency-injection.md`

- [ ] **Step 1: Write SKILL.md**

Create `skills/architecture-patterns/SKILL.md` with:
- Frontmatter: `name: architecture-patterns`, verbose description from spec line 249
- Body:
  1. Title: `# Flutter Architecture Patterns`
  2. Architecture overview: three-layer diagram (Presentation → Domain → Data) with dependency rule
  3. Reference table: 3 rows
  4. Quick Reference: layer responsibilities, dependency direction, what goes where
  5. Anti-Patterns: circular dependencies between layers, domain importing data layer, BLoC knowing about widgets

- [ ] **Step 2: Write clean-architecture.md**

New content. Cover:
- Layer separation: Presentation (widgets + BLoC), Domain (entities + repository interfaces), Data (repo impls + sources + DTOs)
- Dependency rule: inner layers don't know outer layers
- Package structure: `packages/domain/[feature]/lib/`, `packages/data/[feature]/lib/`
- What goes in each layer with decision flowchart
- Cross-layer communication: only through interfaces defined in domain
- Example: full feature structure showing all layers
- Anti-patterns: domain depending on Flutter, data layer exposing implementation details

- [ ] **Step 3: Write feature-driven-development.md**

New content. Cover:
- Feature-first directory structure vs layer-first (and why feature-first wins at scale)
- Feature isolation: each feature is self-contained with its own BLoC, screens, widgets, repository
- Shared kernel: cross-feature code (theme, common widgets, base classes)
- Monorepo with Melos: workspace setup, package dependencies, scripts
- Feature container pattern for DI (from local di-reference.md lines 29-45)
- When to split a feature into sub-features

- [ ] **Step 4: Write dependency-injection.md**

Seed from local `di-reference.md`. Include:
- Pure DI principle: no service locator (Global Rule 2)
- CompositionRoot pattern (from local lines 93-128)
- Dependencies container class (from local lines 16-26)
- Feature containers for logical grouping (from local lines 29-45)
- CoreDependenciesBuilder / FeatureDependenciesBuilder (from local lines 66-91)
- AppRunner initialization (from local lines 141-168)
- AppScope with MultiProvider (from local lines 178-204)
- context.di / context.featureADependencies extensions (from local lines 209-218)
- Anti-pattern: GetIt, injectable, any service locator (with explanation why — from local lines 222-229)
- Add explicit Global Rule 2 callout

- [ ] **Step 5: Commit**

```bash
git add skills/architecture-patterns/
git commit -m "feat: add architecture-patterns skill (DI seeded from local flutter-developer)"
```

### Task 6: animation-performance skill (seeded, split)

**Files:**
- Create: `skills/animation-performance/SKILL.md`
- Create: `skills/animation-performance/references/animation-controllers.md`
- Create: `skills/animation-performance/references/custom-painter.md`
- Create: `skills/animation-performance/references/shaders-render-objects.md`
- Create: `skills/animation-performance/references/performance-checklist.md`

**Source to read first:**
- `.claude/skills/flutter-developer/references/animation-rules.md` — Seeds `animation-controllers.md` (sections 1-2, 10) and `custom-painter.md` (sections 3-9)

**Split boundary (from spec lines 465-468):**
- Sections 1-2, 10 → `animation-controllers.md` (setup and driving animations)
- Sections 3-9 → `custom-painter.md` (painting and rendering efficiently)

- [ ] **Step 1: Write SKILL.md**

Create `skills/animation-performance/SKILL.md` with:
- Frontmatter: `name: animation-performance`, verbose description from spec line 279
- Body:
  1. Title: `# Animation & Performance`
  2. Architecture overview: animation lifecycle (Ticker → Controller → Tween → Widget/Painter)
  3. Reference table: 4 rows
  4. Quick Reference: rules from spec (FPS, Rebuilds, Repaint isolation, Widget structure, CustomPainter, Paint objects, Draw calls, Shaders, Tickers, RenderObject)
  5. Common Pitfalls: 5 items from local animation-rules.md lines 403-407

- [ ] **Step 2: Write animation-controllers.md**

Seed from local `animation-rules.md` sections 1-2 and 10. Include:
- Refresh rate awareness: let Flutter manage it (from local lines 7-17)
- Avoid widget tree rebuilds: AnimatedBuilder over setState (from local lines 21-39)
- Implicit animations: AnimatedContainer, AnimatedOpacity for simple cases (from local lines 299-316)
- Explicit animations: AnimationController + CurvedAnimation + Tween (from local lines 322-371)
- Add: staggered animations (Interval with multiple Tweens)
- Add: Hero animations and shared element transitions
- Add: Rive and Lottie integration basics
- Add: dispose lifecycle — always dispose controllers

- [ ] **Step 3: Write custom-painter.md**

Seed from local `animation-rules.md` sections 3-9. Include:
- RepaintBoundary isolation (from local lines 43-57)
- Structural integrity during animation (from local lines 61-74)
- CustomPainter pattern: repaint param, never setState (from local lines 262-295)
- Paint object reuse (from local lines 168-190)
- Batch drawing: drawPoints, drawAtlas (from local lines 194-208)
- Canvas save/restore (from local lines 212-224)
- Add Global Rule 10 callout

- [ ] **Step 4: Write shaders-render-objects.md**

Seed from local `animation-rules.md` sections 5-6, 8. Include:
- Custom Ticker for precise frame control (from local lines 79-103)
- Direct RenderObject manipulation with markNeedsPaint (from local lines 108-160)
- Fragment shaders: loading, ShaderPainter, uniforms (from local lines 230-258)
- LeafRenderObjectWidget pattern
- Add: when to use each technique (decision guide)

- [ ] **Step 5: Write performance-checklist.md**

New content (compiled from all animation rules + additional). Cover:
- Full checklist format (✅ do / ❌ don't)
- RepaintBoundary isolation checklist
- Animation setup checklist (dispose, vsync, no Timer.periodic)
- CustomPainter checklist (repaint param, Paint reuse, batch ops)
- Widget rebuild checklist (const, keys, buildWhen)
- Performance monitoring: Timeline API, DevTools profiling workflow
- Common pitfalls with severity ratings

- [ ] **Step 6: Commit**

```bash
git add skills/animation-performance/
git commit -m "feat: add animation-performance skill (seeded from local flutter-developer, split per spec)"
```

---

## Chunk 3: New Skills (5 skills written from scratch)

These skills have no local seed content. Write entirely from spec descriptions and Flutter expertise.

### Task 7: testing-strategies skill

**Files:**
- Create: `skills/testing-strategies/SKILL.md`
- Create: `skills/testing-strategies/references/unit-testing.md`
- Create: `skills/testing-strategies/references/widget-testing.md`
- Create: `skills/testing-strategies/references/golden-file-testing.md`
- Create: `skills/testing-strategies/references/integration-testing.md`

- [ ] **Step 1: Write SKILL.md**

Frontmatter: `name: testing-strategies`, description from spec line 289.
Body:
1. Title: `# Flutter Testing Strategies`
2. Testing pyramid: Unit (BLoC, mappers) → Widget (UI components) → Golden (visual regression) → Integration (E2E)
3. Reference table: 4 rows
4. Quick Reference: test type → what to test → tools → when to use
5. Anti-Patterns: testing implementation details, mocking too much, no golden updates in CI

- [ ] **Step 2: Write unit-testing.md**

Cover:
- bloc_test package: blocTest() with build/act/expect pattern
- Testing sealed state sequences (emit order matters)
- Testing transformer behavior: verify droppable drops, restartable cancels, sequential queues
- Mockito setup: @GenerateMocks for repository interfaces
- Testing domain exceptions (expect specific exception types)
- Testing mappers (DTO → Entity → DTO roundtrip)
- Example: complete BLoC test file for a feature

- [ ] **Step 3: Write widget-testing.md**

Cover:
- testWidgets setup with pumpWidget
- Wrapping with BlocProvider and MaterialApp for theme
- Finding widgets: find.byType, find.text, find.byKey
- Tapping, entering text, scrolling
- Verifying BlocBuilder renders correct widget per sealed state (all branches)
- pump() vs pumpAndSettle() — when to use each
- Testing BlocListener side effects (verify navigation, snackbar)
- Example: complete widget test for a screen with BlocBuilder

- [ ] **Step 4: Write golden-file-testing.md**

Cover:
- Golden file concept: snapshot-based visual regression testing
- Setup: matchesGoldenFile, font loading (FontLoader) for deterministic renders
- Creating goldens: `flutter test --update-goldens`
- CI configuration: running golden tests, handling platform differences
- Platform-specific golden management (separate goldens per OS)
- Golden file naming conventions
- When to update goldens vs when a test failure is a real regression

- [ ] **Step 5: Write integration-testing.md**

Cover:
- Patrol setup: pubspec, patrol.yaml, PatrolTester
- Native interaction testing: system dialogs, permissions, notifications
- integration_test package basics: IntegrationTestWidgetsFlutterBinding
- Test drivers for CI execution
- Running on device farms (Firebase Test Lab, AWS Device Farm)
- CI integration: GitHub Actions workflow for integration tests
- When to write integration tests vs widget tests

- [ ] **Step 6: Commit**

```bash
git add skills/testing-strategies/
git commit -m "feat: add testing-strategies skill"
```

### Task 8: platform-integration skill

**Files:**
- Create: `skills/platform-integration/SKILL.md`
- Create: `skills/platform-integration/references/platform-channels.md`
- Create: `skills/platform-integration/references/ffi-native-code.md`
- Create: `skills/platform-integration/references/platform-specific-features.md`

- [ ] **Step 1: Write SKILL.md**

Frontmatter: `name: platform-integration`, description from spec line 299.
Body:
1. Title: `# Platform Integration`
2. Architecture overview: Flutter ↔ Platform Channel ↔ Native Code
3. Reference table: 3 rows
4. Quick Reference: channel type → use case → direction → example
5. Anti-Patterns: using platform channels for everything (use packages first), no error handling across boundary

- [ ] **Step 2: Write platform-channels.md**

Cover:
- MethodChannel: request/response pattern, Dart + Swift + Kotlin examples
- EventChannel: stream from native to Dart, use cases (sensors, location)
- BasicMessageChannel: simple bidirectional messaging
- Codec selection: StandardMessageCodec (default), JSONMessageCodec
- Error handling: PlatformException, MissingPluginException
- Platform-specific implementation pattern (switch on Platform.isIOS etc.)
- Testing platform channels with TestDefaultBinaryMessenger

- [ ] **Step 3: Write ffi-native-code.md**

Cover:
- Dart FFI basics: DynamicLibrary, NativeFunction, lookupFunction
- ffigen for automatic binding generation from C headers
- Memory management: malloc/free, Pointer types, Finalizer
- Struct passing: @Struct annotation, field access
- Callbacks: NativeCallable.listener for async callbacks from C
- package:ffi utilities
- When to use FFI vs platform channels (decision guide)

- [ ] **Step 4: Write platform-specific-features.md**

Cover per platform:
- iOS: Face ID/Touch ID (local_auth), Haptic feedback (HapticFeedback), Live Activities, App Clips, WidgetKit
- Android: BiometricPrompt, Home screen widgets, Material You dynamic theming, App Links
- Web: PWA manifest and service worker, JS interop with dart:js_interop, web-specific rendering considerations
- Desktop: System tray, menu bar integration, window management (window_manager), file system access, keyboard shortcuts
- Cross-platform pattern: abstract interface + platform-specific implementations

- [ ] **Step 5: Commit**

```bash
git add skills/platform-integration/
git commit -m "feat: add platform-integration skill"
```

### Task 9: dart-language-mastery skill

**Files:**
- Create: `skills/dart-language-mastery/SKILL.md`
- Create: `skills/dart-language-mastery/references/patterns-records-sealed.md`
- Create: `skills/dart-language-mastery/references/async-isolates.md`
- Create: `skills/dart-language-mastery/references/code-generation.md`

- [ ] **Step 1: Write SKILL.md**

Frontmatter: `name: dart-language-mastery`, description from spec line 308.
Body:
1. Title: `# Dart Language Mastery`
2. Overview: Dart 3.x features that power Flutter architecture
3. Reference table: 3 rows
4. Quick Reference: feature → syntax → when to use
5. Anti-Patterns: avoiding Dart 3 features (patterns, sealed), using dynamic/Object where sealed classes fit

- [ ] **Step 2: Write patterns-records-sealed.md**

Cover:
- Pattern matching: switch expressions with exhaustiveness checking
- Destructuring: positional (`var (a, b) = record`) and named
- Records: multiple return values, type-safe tuples
- Sealed class hierarchies: for type-safe unions (states, events, results)
- Guard clauses in patterns: `case Foo(:final x) when x > 0`
- Logical patterns: `&&`, `||` in case clauses
- Relational patterns: `case >= 0 && <= 100`
- Real-world Flutter examples: sealed states in BlocBuilder switch

- [ ] **Step 3: Write async-isolates.md**

Cover:
- Future and async/await best practices
- Stream types: single-subscription vs broadcast
- StreamController: creating custom streams, close lifecycle
- Isolate.run() for CPU-intensive work (JSON parsing, image processing)
- compute() shorthand (deprecated in favor of Isolate.run)
- Zone error handling: Zone.current.handleUncaughtError
- Stream transformers: map, where, asyncMap, asyncExpand
- Completer: bridging callback-based APIs to Future

- [ ] **Step 4: Write code-generation.md**

Cover:
- build_runner setup: pubspec dev dependencies, build.yaml configuration
- json_serializable: @JsonSerializable, @JsonKey, FieldRename.snake, fromJson/toJson
- freezed: documented as alternative to hand-written sealed+Equatable — NOT recommended for BLoC states/events, acceptable for complex DTOs or value objects outside BLoC layer
- go_router: route code generation with type-safe paths (cross-reference to navigation.md)
- injectable: ANTI-PATTERN — documented only to flag. Uses get_it service locator, violates Global Rule 2
- Custom Builder implementation basics
- Running: `dart run build_runner build --delete-conflicting-outputs`

- [ ] **Step 5: Commit**

```bash
git add skills/dart-language-mastery/
git commit -m "feat: add dart-language-mastery skill"
```

### Task 10: devops-deployment skill

**Files:**
- Create: `skills/devops-deployment/SKILL.md`
- Create: `skills/devops-deployment/references/flavors-environments.md`
- Create: `skills/devops-deployment/references/cicd-pipelines.md`
- Create: `skills/devops-deployment/references/app-store-deployment.md`

- [ ] **Step 1: Write SKILL.md**

Frontmatter: `name: devops-deployment`, description from spec line 317.
Body:
1. Title: `# DevOps & Deployment`
2. Overview: build → test → deploy pipeline for multi-platform Flutter
3. Reference table: 3 rows
4. Quick Reference: environment → flavor → bundle ID → API endpoint
5. Anti-Patterns: hardcoded API URLs, no flavor separation, manual signing

- [ ] **Step 2: Write flavors-environments.md**

Cover:
- Flutter flavors: dev/staging/prod configuration
- --dart-define and --dart-define-from-file for build-time variables
- Per-flavor configuration: bundle IDs, app names, icons
- Android productFlavors in build.gradle
- iOS schemes and configurations in Xcode
- Environment-specific API endpoints pattern
- Flavor-aware launcher icons (flutter_launcher_icons per flavor)

- [ ] **Step 3: Write cicd-pipelines.md**

Cover:
- GitHub Actions workflow: complete YAML for Flutter (checkout → setup → test → build → deploy)
- Codemagic: codemagic.yaml configuration
- Bitrise: bitrise.yml basics
- Caching: pub cache, build artifacts
- Automated testing in CI: unit + widget + golden
- Build artifact publishing (APK, IPA, web)
- Version bumping automation (pubspec.yaml version)
- Branch strategy: feature → develop → main

- [ ] **Step 4: Write app-store-deployment.md**

Cover:
- iOS: provisioning profiles, certificates, match (fastlane), TestFlight, App Store Connect submission, review guidelines
- Google Play: upload key vs app signing key, Play Console tracks (internal/alpha/beta/production), staged rollout
- Web: Firebase Hosting, Vercel, Cloudflare Pages deployment
- Desktop: DMG packaging (macOS), MSI/MSIX (Windows), AppImage/Snap (Linux), code signing, notarization (macOS)
- Automation: fastlane for iOS/Android, GitHub Actions for web

- [ ] **Step 5: Commit**

```bash
git add skills/devops-deployment/
git commit -m "feat: add devops-deployment skill"
```

### Task 11: security-compliance skill

**Files:**
- Create: `skills/security-compliance/SKILL.md`
- Create: `skills/security-compliance/references/secure-storage.md`
- Create: `skills/security-compliance/references/network-security.md`
- Create: `skills/security-compliance/references/compliance-gdpr.md`

- [ ] **Step 1: Write SKILL.md**

Frontmatter: `name: security-compliance`, description from spec line 326.
Body:
1. Title: `# Security & Compliance`
2. Overview: security layers (storage → network → app → compliance)
3. Reference table: 3 rows
4. Quick Reference: concern → solution → package
5. Anti-Patterns: storing tokens in SharedPreferences (use SecureStorage), hardcoded API keys, no certificate pinning

- [ ] **Step 2: Write secure-storage.md**

Cover:
- flutter_secure_storage: setup, read/write/delete, options per platform
- iOS Keychain: accessibility options, keychain sharing
- Android KeyStore: EncryptedSharedPreferences, hardware-backed keys
- Token lifecycle: access token storage, refresh token rotation, expiry handling
- Encryption at rest for sensitive local data
- Secure deletion: overwriting before deleting
- When to use SecureStorage vs SharedPreferences decision guide

- [ ] **Step 3: Write network-security.md**

Cover:
- Certificate pinning: Dio interceptor with SHA-256 pin comparison
- HTTPS enforcement: no HTTP fallback
- API key management: environment variables, --dart-define, never in source code
- Request signing: HMAC for API integrity
- Android network security config: res/xml/network_security_config.xml
- iOS App Transport Security: Info.plist configuration
- SSL/TLS version requirements

- [ ] **Step 4: Write compliance-gdpr.md**

Cover:
- GDPR principles: lawful basis, data minimization, purpose limitation
- Privacy-by-design: collect only necessary data
- User consent management: consent dialog, preference storage, opt-out
- Right to deletion: API endpoint + local data cleanup
- Data export: user data portability
- --obfuscate and --split-debug-info for release builds
- Audit logging: what to log, what never to log (PII)
- Cookie consent for Flutter web

- [ ] **Step 5: Commit**

```bash
git add skills/security-compliance/
git commit -m "feat: add security-compliance skill"
```

---

## Chunk 4: Agents (8 agent files)

All agents follow the format defined in the spec's "Agent File Format" section (lines 134-157). Each agent is 150-310 lines. Read the backend-development agents for format examples before writing.

**Reference:** Read `~/.claude/plugins/cache/claude-plugins-official/backend-development/1.3.1/agents/backend-architect.md` as the format template before writing any agent.

**All agents MUST include in Behavioral Traits:** The relevant subset of the 10 Global Rules from spec lines 123-132.

### Task 12: flutter-architect agent

**Files:**
- Create: `agents/flutter-architect.md`

- [ ] **Step 1: Write flutter-architect.md**

Frontmatter:
```yaml
---
name: flutter-architect
description: Expert Flutter architect specializing in Clean Architecture, feature-driven development, and dependency injection. Masters layer separation, module boundaries, package structure, and CompositionRoot patterns. Use PROACTIVELY when creating new features, restructuring projects, or discussing module boundaries.
model: inherit
---
```

Body sections (target ~200 lines):
1. **Purpose**: Expert Flutter architect for Clean Architecture, feature-driven development, DI with CompositionRoot
2. **Capabilities**: subsections for Architecture Patterns, Package Structure, DI Patterns, Module Boundaries, Monorepo Management
3. **Behavioral Traits**: 10 items including Global Rules 1-5 (BLoC only, constructor injection, no DTO exposure, sealed classes, explicit transformers)
4. **Knowledge Base**: Flutter architecture ecosystem, Melos, package conventions
5. **Response Approach**: 8-step methodology (analyze requirements → define layers → design DI → propose structure → review boundaries → document decisions)
6. **Example Interactions**: 6-8 sample prompts

- [ ] **Step 2: Commit**

```bash
git add agents/flutter-architect.md
git commit -m "feat: add flutter-architect agent"
```

### Task 13: flutter-state-expert agent

**Files:**
- Create: `agents/flutter-state-expert.md`

- [ ] **Step 1: Write flutter-state-expert.md**

Frontmatter:
```yaml
---
name: flutter-state-expert
description: Expert in BLoC state management with sealed classes, event transformers, and race condition prevention. Masters event-driven architecture, transformer selection, and BLoC-to-BLoC communication via repository streams. Use PROACTIVELY when implementing business logic, discussing state shape, or debugging race conditions.
model: inherit
---
```

Body (~250 lines, this is a core agent):
1. **Purpose**: BLoC state management expert, race condition prevention, transformer strategy
2. **Capabilities**: subsections for BLoC Core, State Design, Event Design, Transformer Strategies, BLoC Communication, Stream Patterns
3. **Behavioral Traits**: 10 items. MUST include all of: Global Rule 1 (never Cubit), Rule 2 (constructor injection), Rule 4 (sealed+Equatable), Rule 5 (explicit transformers)
4. **Knowledge Base**: bloc_concurrency, flutter_bloc, equatable ecosystem
5. **Response Approach**: analyze state shape → design events → select transformers → implement → verify race conditions
6. **Example Interactions**: 8 samples covering BLoC creation, debugging, transformer selection

- [ ] **Step 2: Commit**

```bash
git add agents/flutter-state-expert.md
git commit -m "feat: add flutter-state-expert agent"
```

### Task 14: flutter-ui-expert agent

**Files:**
- Create: `agents/flutter-ui-expert.md`

- [ ] **Step 1: Write flutter-ui-expert.md**

Frontmatter:
```yaml
---
name: flutter-ui-expert
description: Expert in Flutter widget composition, BlocBuilder/Listener/Consumer patterns, responsive design, theming, and navigation. Masters Material 3, Cupertino design, and adaptive layouts. Use PROACTIVELY when building screens, connecting UI to BLoC, or implementing design systems.
model: sonnet
---
```

Body (~200 lines):
1. **Purpose**: Widget composition, BLoC-UI integration, responsive/adaptive design
2. **Capabilities**: Widget Composition, BLoC Widgets, Responsive/Adaptive, Navigation (go_router), Theming (Material 3)
3. **Behavioral Traits**: Global Rules 7 (Theme.of), 8 (localization), 9 (StatelessWidget default), plus widget-specific principles
4. **Knowledge Base**: Material 3, Cupertino, go_router, responsive patterns
5. **Response Approach**: analyze screen requirements → compose widgets → wire BLoC → add responsiveness → apply theme
6. **Example Interactions**: 6 samples

- [ ] **Step 2: Commit**

```bash
git add agents/flutter-ui-expert.md
git commit -m "feat: add flutter-ui-expert agent"
```

### Task 15: flutter-animation-expert agent

**Files:**
- Create: `agents/flutter-animation-expert.md`

- [ ] **Step 1: Write flutter-animation-expert.md**

Frontmatter:
```yaml
---
name: flutter-animation-expert
description: Expert in Flutter animations, CustomPainter, fragment shaders, and rendering performance. Masters AnimationController, Impeller optimization, RenderObject manipulation, and batch drawing techniques. Use PROACTIVELY when implementing animations or diagnosing rendering performance.
model: sonnet
---
```

Body (~200 lines):
1. **Purpose**: Animation implementation, CustomPainter, GPU shaders, render optimization
2. **Capabilities**: Animation Controllers, CustomPainter, Shaders, RenderObjects, Impeller
3. **Behavioral Traits**: Global Rule 10 (RepaintBoundary, never setState for CustomPainter), plus animation-specific
4. **Knowledge Base**: Impeller, Skia legacy, fragment shader GLSL, DevTools timeline
5. **Response Approach**: analyze animation requirements → select technique → implement → profile → optimize
6. **Example Interactions**: 6 samples

- [ ] **Step 2: Commit**

```bash
git add agents/flutter-animation-expert.md
git commit -m "feat: add flutter-animation-expert agent"
```

### Task 16: flutter-testing-expert agent

**Files:**
- Create: `agents/flutter-testing-expert.md`

- [ ] **Step 1: Write flutter-testing-expert.md**

Frontmatter:
```yaml
---
name: flutter-testing-expert
description: Expert in Flutter testing including unit tests with bloc_test, widget tests, golden file testing, and integration tests with Patrol. Masters TDD red-green-refactor workflow and comprehensive test coverage. Use PROACTIVELY when implementing features (TDD), reviewing untested code, or setting up test infrastructure.
model: sonnet
---
```

Body (~200 lines):
1. **Purpose**: TDD for Flutter, comprehensive test coverage, testing sealed state branches
2. **Capabilities**: Unit Testing (BLoC), Widget Testing, Golden Files, Integration Testing (Patrol), Coverage Analysis
3. **Behavioral Traits**: TDD discipline, test all sealed state branches, mock at boundary not internally
4. **Knowledge Base**: bloc_test, mockito, golden_toolkit, patrol, integration_test
5. **Response Approach**: write failing test → implement minimal code → run tests → refactor → commit
6. **Example Interactions**: 6 samples

- [ ] **Step 2: Commit**

```bash
git add agents/flutter-testing-expert.md
git commit -m "feat: add flutter-testing-expert agent"
```

### Task 17: flutter-performance-engineer agent

**Files:**
- Create: `agents/flutter-performance-engineer.md`

- [ ] **Step 1: Write flutter-performance-engineer.md**

Frontmatter:
```yaml
---
name: flutter-performance-engineer
description: Expert in Flutter performance profiling, widget rebuild optimization, memory management, and bundle size reduction. Masters DevTools, Isolates, Impeller migration, and frame rendering optimization. Use PROACTIVELY when reviewing code for performance issues or diagnosing runtime problems.
model: sonnet
---
```

Body (~180 lines):
1. **Purpose**: Performance profiling, rebuild optimization, memory management
2. **Capabilities**: DevTools Profiling, Rebuild Optimization, Memory Management, Bundle Size, Impeller
3. **Behavioral Traits**: measure before optimizing, const constructors, Isolate for CPU work
4. **Knowledge Base**: DevTools, Impeller, tree shaking, deferred loading
5. **Response Approach**: profile → identify bottleneck → measure baseline → optimize → verify improvement
6. **Example Interactions**: 6 samples

- [ ] **Step 2: Commit**

```bash
git add agents/flutter-performance-engineer.md
git commit -m "feat: add flutter-performance-engineer agent"
```

### Task 18: flutter-platform-engineer agent

**Files:**
- Create: `agents/flutter-platform-engineer.md`

- [ ] **Step 1: Write flutter-platform-engineer.md**

Frontmatter:
```yaml
---
name: flutter-platform-engineer
description: Expert in Flutter platform channels, Dart FFI, and platform-specific feature implementation across iOS, Android, Web, and Desktop. Masters native plugin development, bidirectional communication, and platform-specific integrations. Use PROACTIVELY when adding native features or building platform-specific functionality.
model: inherit
---
```

Body (~220 lines):
1. **Purpose**: Platform channel architecture, FFI, native integration across all platforms
2. **Capabilities**: Platform Channels (Method/Event/Basic), FFI, iOS Integration (Swift), Android Integration (Kotlin), Web (JS interop), Desktop
3. **Behavioral Traits**: prefer existing packages first, graceful degradation, error handling at boundary
4. **Knowledge Base**: iOS/Android/Web/Desktop APIs, ffigen, platform-specific SDKs
5. **Response Approach**: check if package exists → design channel interface → implement Dart side → implement native side → test across platforms
6. **Example Interactions**: 6 samples

- [ ] **Step 2: Commit**

```bash
git add agents/flutter-platform-engineer.md
git commit -m "feat: add flutter-platform-engineer agent"
```

### Task 19: flutter-data-engineer agent

**Files:**
- Create: `agents/flutter-data-engineer.md`

- [ ] **Step 1: Write flutter-data-engineer.md**

Frontmatter:
```yaml
---
name: flutter-data-engineer
description: Expert in Flutter data layer architecture with repositories, Drift persistence, Dio/Retrofit API integration, and offline-first patterns. Masters entity/DTO separation, mapper patterns, domain exceptions, and caching strategies. Use PROACTIVELY when designing data flow, implementing repositories, or setting up persistence.
model: sonnet
---
```

Body (~220 lines):
1. **Purpose**: Data layer architecture, repository pattern, persistence, API integration
2. **Capabilities**: Entity/DTO Design, Repository Pattern, Drift, API Integration (Dio/Retrofit), Offline-First, Caching, Mappers
3. **Behavioral Traits**: Global Rule 3 (never expose DTOs), Rule 6 (no Hive), map at boundary, domain exceptions
4. **Knowledge Base**: Drift, Dio, Retrofit, json_serializable, equatable
5. **Response Approach**: define entities → design repository interface → implement data sources → build repository → map exceptions → add caching
6. **Example Interactions**: 6 samples

- [ ] **Step 2: Commit**

```bash
git add agents/flutter-data-engineer.md
git commit -m "feat: add flutter-data-engineer agent"
```

---

## Chunk 5: Orchestration Command + Final Verification

### Task 20: flutter-feature orchestration command

**Files:**
- Create: `commands/flutter-feature.md`

**Reference:** Read `~/.claude/plugins/cache/claude-plugins-official/backend-development/1.3.1/commands/feature-development.md` as the format template. The Flutter command must match this level of detail (~500 lines).

This is the most complex file in the plugin (~500 lines). It must be a complete, executable command document matching the format of backend-development's `feature-development.md`. Split into 3 write steps.

Each step's Agent prompt must:
- Reference the agent by `subagent_type` (e.g., `flutter-architect`)
- Include the feature description variable (`$FEATURE`)
- Reference prior step outputs with "Insert contents of .flutter-dev/..." instructions
- Specify deliverables expected from the agent
- End with "Write your complete [output type] as a single markdown document."

- [ ] **Step 1: Write frontmatter + behavioral rules + pre-flight checks**

Create `commands/flutter-feature.md` with:

```yaml
---
description: "Orchestrate end-to-end Flutter feature development from requirements to deployment"
argument-hint: "<feature description> [--architecture clean|feature-driven] [--complexity simple|medium|complex]"
---
```

Then write:
1. **Critical Behavioral Rules** (6 rules from spec lines 430-435)
2. **Pre-flight Checks**: check `.flutter-dev/state.json` for existing session (resume or start fresh), initialize state.json with feature/architecture/complexity/current_step, parse `$ARGUMENTS` for `$FEATURE` and flags

- [ ] **Step 2: Write Phase 1 discovery + Checkpoint 1 + Phase 2 implementation + Checkpoint 2**

Append to `commands/flutter-feature.md`:

**Phase 1: Discovery (Steps 1-3)** — Sequential. Each step includes:
- Full Agent tool call block with `subagent_type`, `description`, and multi-line `prompt`
- Output file path (`.flutter-dev/0N-*.md`)
- State update instructions (current_step, completed_steps)

Steps:
- Step 1: flutter-architect → `.flutter-dev/01-architecture.md`
- Step 2: flutter-state-expert (reads Step 1) → `.flutter-dev/02-state-design.md`
- Step 3: flutter-data-engineer (reads Steps 1-2) → `.flutter-dev/03-data-design.md`

**PHASE CHECKPOINT 1** — AskUserQuestion with 3 options: approve, request changes, pause

**Phase 2: Implementation (Steps 4a-6)** — Parallel where noted:
- 4a (flutter-state-expert) + 4b (flutter-data-engineer) in parallel
- 4c (flutter-ui-expert) depends on 4a+4b
- 5 (flutter-testing-expert) + 6 (flutter-performance-engineer) in parallel, depend on 4a-4c

**PHASE CHECKPOINT 2** — AskUserQuestion with 3 options

- [ ] **Step 3: Write Phase 3 delivery + completion summary**

Append to `commands/flutter-feature.md`:

**Phase 3: Delivery (Step 7)** — Sequential:
- Step 7: flutter-platform-engineer (reads all prior) → `.flutter-dev/07-platform.md`

**Completion**:
- Update state.json: status "complete", last_updated timestamp
- Final summary: list all files, implementation summary per phase, success criteria

- [ ] **Step 4: Commit**

```bash
git add commands/flutter-feature.md
git commit -m "feat: add flutter-feature orchestration command"
```

### Task 21: Final verification and release commit

- [ ] **Step 1: Verify directory structure matches spec**

Run from plugin root:
```bash
find flutter-crossplatform-development/ -type f | sort
```

Expected: 56 files total. Verify against spec directory tree (lines 23-104).

- [ ] **Step 2: Verify all SKILL.md files have correct frontmatter**

Check each SKILL.md has `name` and `description` in frontmatter:
```bash
for f in flutter-crossplatform-development/skills/*/SKILL.md; do head -5 "$f"; echo "---"; done
```

- [ ] **Step 3: Verify all agent files have correct frontmatter**

Check each agent has `name`, `description`, and `model` in frontmatter:
```bash
for f in flutter-crossplatform-development/agents/*.md; do head -5 "$f"; echo "---"; done
```

- [ ] **Step 4: Verify command file has correct frontmatter**

```bash
head -5 flutter-crossplatform-development/commands/flutter-feature.md
```

- [ ] **Step 5: Final commit**

```bash
git add flutter-crossplatform-development/
git commit -m "feat: flutter-crossplatform-development plugin v1.0.0 complete

8 agents, 10 skills (36 references), 1 orchestration command.
Opinionated Flutter development: BLoC-only, Clean Architecture,
sealed classes, constructor injection, explicit transformers."
```

---

## Parallelization Guide

For subagent-driven execution, these task groups can run in parallel:

| Parallel Group | Tasks | Dependency |
|---|---|---|
| Group 1 | Task 1 (scaffold) | None — must run first |
| Group 2 | Tasks 2, 3, 4, 5, 6 (seeded skills) | After Task 1 |
| Group 3 | Tasks 7, 8, 9, 10, 11 (new skills) | After Task 1 |
| Group 4 | Tasks 12-19 (all 8 agents) | After Task 1 |
| Group 5 | Task 20 (command) | After Task 1 (reads agents for subagent_type names) |
| Group 6 | Task 21 (verification) | After all other tasks |

**Maximum parallelism:** Groups 2, 3, 4, and 5 can all run simultaneously after Group 1 completes. This means up to 19 tasks can run in parallel.
