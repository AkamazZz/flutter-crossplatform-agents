---
description: "Orchestrate end-to-end Flutter feature development from requirements to deployment"
argument-hint: "<feature description> [--architecture clean|feature-driven] [--complexity simple|medium|complex]"
---

# Flutter Feature Development Orchestrator

## CRITICAL BEHAVIORAL RULES

You MUST follow these rules exactly. Violating any of them is a failure.

1. **Execute steps in order.** Do NOT skip ahead, reorder, or merge steps.
2. **Write output files.** Each step MUST produce its output file in `.flutter-dev/` before the next step begins. Read from prior step files — do NOT rely on context window memory.
3. **Stop at checkpoints.** When you reach a `PHASE CHECKPOINT`, you MUST stop and wait for explicit user approval before continuing. Use the AskUserQuestion tool with clear options.
4. **Halt on failure.** If any step fails (agent error, test failure, missing dependency), STOP immediately. Present the error and ask the user how to proceed. Do NOT silently continue.
5. **Use only plugin-local agents.** All `subagent_type` references use agents bundled with this plugin. No cross-plugin dependencies.
6. **Never enter plan mode autonomously.** Do NOT use EnterPlanMode. This command IS the plan — execute it.

## Pre-flight Checks

Before starting, perform these checks:

### 1. Check for existing session

Check if `.flutter-dev/state.json` exists:

- If it exists and `status` is `"in_progress"`: Read it, display the current step, and ask the user:

  ```
  Found an in-progress Flutter feature development session:
  Feature: [name from state]
  Current step: [step from state]
  Phase: [phase from state]

  1. Resume from where we left off
  2. Start fresh (archives existing session)
  ```

- If it exists and `status` is `"complete"`: Ask whether to archive and start fresh.

### 2. Initialize state

Create `.flutter-dev/` directory and `state.json`:

```json
{
  "feature": "$ARGUMENTS",
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

Parse `$ARGUMENTS` for `--architecture` and `--complexity` flags. Use defaults (`clean` and `medium`) if not specified.

### 3. Parse feature description

Extract the feature description from `$ARGUMENTS` (everything before the flags). This is referenced as `$FEATURE` in prompts below.

---

## Phase 1: Discovery (Steps 1–3) — Sequential

### Step 1: Requirements & Architecture

Use the Task tool to launch the flutter architect:

```
Task:
  subagent_type: "flutter-architect"
  description: "Design architecture for $FEATURE"
  prompt: |
    Design the architecture for this Flutter feature: $FEATURE

    ## Architecture Style
    [Insert architecture value from state.json — "clean" or "feature-driven"]

    ## Complexity
    [Insert complexity value from state.json — "simple", "medium", or "complex"]

    ## Instructions
    Analyze the feature requirements and produce a complete architecture design:

    1. **Feature Scope Analysis** — Break down $FEATURE into concrete requirements.
       Identify user-facing screens, background processes, and data flows.

    2. **Clean Architecture Layer Boundaries** — Define exactly which components
       belong to Presentation, Domain, and Data layers. Enforce the dependency rule:
       inner layers MUST NOT know about outer layers.

    3. **Package Structure** — Design the directory layout following the project's
       conventions. Specify where each file goes:
       - `lib/features/$FEATURE/presentation/` — screens, widgets, BLoC classes
       - `lib/features/$FEATURE/domain/` — entities, repository interfaces, use cases
       - `lib/features/$FEATURE/data/` — repository implementations, DTOs, data sources
       Or, if feature-driven architecture is selected, specify the feature-first layout.

    4. **Dependency Injection Plan** — Define what gets registered in the
       CompositionRoot. Specify the Dependencies container class additions,
       FeatureDependenciesBuilder entries, and how the feature container connects
       to AppScope via MultiProvider. Constructor injection only — no service locator.

    5. **Integration Points** — How this feature connects to existing features,
       shared repositories, or navigation routes.

    ## Deliverables
    1. Architecture document with layer diagram showing Presentation → Domain → Data
    2. Complete package/directory structure with every planned file listed
    3. DI plan specifying CompositionRoot additions and dependency wiring
    4. Integration points with existing app infrastructure
    5. Risk assessment and technical considerations

    Write your complete architecture design as a single markdown document.
```

Save the agent's output to `.flutter-dev/01-architecture.md`.

Update `state.json`: set `current_step` to 2, add `"01-architecture.md"` to `files_created`, add step 1 to `completed_steps`, set `last_updated`.

### Step 2: State Management Design

Read `.flutter-dev/01-architecture.md` to load architecture context.

Use the Task tool to launch the state management expert:

```
Task:
  subagent_type: "flutter-state-expert"
  description: "Design BLoC state management for $FEATURE"
  prompt: |
    Design the BLoC state management layer for this Flutter feature: $FEATURE

    ## Architecture
    [Insert full contents of .flutter-dev/01-architecture.md]

    ## Instructions
    Based on the approved architecture, design every BLoC class this feature needs:

    1. **BLoC Class Inventory** — List every BLoC required for $FEATURE. For each BLoC,
       define its single responsibility and the screen/flow it serves. Never combine
       unrelated concerns into one BLoC.

    2. **Sealed Event Classes** — For each BLoC, define the complete sealed event
       hierarchy. Use action-oriented names (e.g., `ProfileLoadRequested`,
       `ProfileUpdateSubmitted`, not `LoadProfile` or `UpdateProfile`). Include all
       event properties with types.

    3. **Sealed State Classes** — For each BLoC, define the complete sealed state
       hierarchy using Equatable. States MUST be immutable with final fields only.
       Include `props` override for every state. Define shared base properties
       (items, cursor, hasMore) where pagination or lists are involved. Add
       `isProcessing` getters where applicable.

    4. **Transformer Selection** — For EVERY `on<Event>` handler, specify the
       explicit transformer:
       - `sequential()` — default for most operations (form submissions, CRUD)
       - `droppable()` — for navigation events, login, one-shot actions
       - `restartable()` — for search/autocomplete, stream subscriptions
       Document WHY each transformer was chosen for each event handler.

    5. **Repository Interfaces** — Define the abstract repository interfaces that
       each BLoC needs injected via constructor. Specify method signatures with
       return types (Entity types, never DTOs). Include `@useResult` annotations.

    6. **BLoC-to-BLoC Communication** — If multiple BLoCs need to share state,
       design the shared repository stream pattern. Never inject one BLoC into
       another.

    ## Rules
    - No Cubit — BLoC only
    - Constructor injection only — no getIt inside BLoC
    - No public methods — only add(), stream, state
    - No Flutter imports in BLoC layer
    - Every on<> handler MUST have an explicit transformer

    ## Deliverables
    1. Complete BLoC specifications with events, states, and transformer assignments
    2. Repository interface definitions with method signatures
    3. BLoC-to-BLoC communication design (if applicable)
    4. State flow diagrams showing event → handler → state transitions

    Write your complete state management design as a single markdown document.
```

Save the agent's output to `.flutter-dev/02-state-design.md`.

Update `state.json`: set `current_step` to 3, add `"02-state-design.md"` to `files_created`, add step 2 to `completed_steps`, set `last_updated`.

### Step 3: Data Layer Design

Read `.flutter-dev/01-architecture.md` and `.flutter-dev/02-state-design.md`.

Use the Task tool to launch the data engineer:

```
Task:
  subagent_type: "flutter-data-engineer"
  description: "Design data layer for $FEATURE"
  prompt: |
    Design the complete data layer for this Flutter feature: $FEATURE

    ## Architecture
    [Insert full contents of .flutter-dev/01-architecture.md]

    ## State Management Design
    [Insert full contents of .flutter-dev/02-state-design.md]

    ## Instructions
    Based on the architecture and state management design, define every data layer
    component this feature needs:

    1. **Domain Entities** — Define pure Dart entity classes with Equatable. No JSON
       serialization, no Flutter imports, no business logic in entities. Include
       `props` override and `copyWith` methods. These are the types that BLoC states
       hold and repository interfaces return.

    2. **DTOs (Data Transfer Objects)** — Define DTO classes with `json_serializable`
       and `FieldRename.snake`. DTOs mirror the API response shape. Include
       `fromJson`/`toJson` factory methods. DTOs MUST NOT leak into the Domain or
       Presentation layers.

    3. **Repository Implementations** — Implement the abstract repository interfaces
       defined in the state management design. Mark as `@immutable`. Inject data
       sources via constructor. Map DTOs to entities at the repository boundary —
       never return DTOs. Catch data-layer exceptions (DioException, etc.) and
       rethrow as domain exceptions.

    4. **API Contracts** — Define the REST endpoints this feature consumes:
       - HTTP method, path, query parameters, request body
       - Response schema (matches DTO structure)
       - Error response format
       - If using Retrofit, define the abstract client interface with annotations.

    5. **Drift Schemas (if applicable)** — If the feature needs local persistence
       beyond simple key-value:
       - Define Drift table classes with column types
       - Define DAOs for organized query access
       - Plan migration strategy if tables are new
       - Define reactive queries with `watch()` where BLoC needs live updates

    6. **Mappers** — Define mapper classes for complex or bidirectional conversions
       between DTOs and entities. Simple one-way mappings can use extension methods
       on the DTO. Inject dependencies into mapper constructors if needed.

    7. **Data Sources** — Define local and remote data source abstractions if the
       repository needs to coordinate between API and local storage (offline-first,
       caching).

    ## Rules
    - No Hive — use Drift for SQL, SharedPreferences/SecureStorage for key-value
    - Entities are pure Dart + Equatable — no JSON, no Flutter
    - DTOs use json_serializable with snake_case field rename
    - Repository implementations are @immutable with constructor-injected sources
    - Domain exceptions at boundary — never expose DioException to BLoC
    - Never expose DTOs to BLoC layer — always map to entities first

    ## Deliverables
    1. Entity class definitions with properties, Equatable props, and copyWith
    2. DTO class definitions with json_serializable annotations
    3. Repository implementation specifications
    4. API contract definitions (endpoints, schemas, error formats)
    5. Drift table schemas and DAOs (if applicable)
    6. Mapper specifications
    7. Data source abstractions (if applicable)

    Write your complete data layer specification as a single markdown document.
```

Save the agent's output to `.flutter-dev/03-data-design.md`.

Update `state.json`: set `current_step` to "checkpoint-1", set `current_phase` to 1, add `"03-data-design.md"` to `files_created`, add step 3 to `completed_steps`, set `last_updated`.

---

## PHASE CHECKPOINT 1 — User Approval Required

You MUST stop here and present the discovery phase for review.

Display a summary from `.flutter-dev/01-architecture.md`, `.flutter-dev/02-state-design.md`, and `.flutter-dev/03-data-design.md` (key architectural decisions, BLoC classes planned, entity/DTO overview, API endpoints) and ask:

```
Architecture and state design complete. Please review:
- .flutter-dev/01-architecture.md
- .flutter-dev/02-state-design.md
- .flutter-dev/03-data-design.md

1. Approve — proceed to implementation
2. Request changes — tell me what to adjust
3. Pause — save progress and stop here
```

Do NOT proceed to Phase 2 until the user selects option 1. If they select option 2, revise the relevant step(s) and re-checkpoint. If option 3, update `state.json` status and stop.

---

## Phase 2: Implementation (Steps 4a–6) — Parallel Where Possible

### Step 4a: BLoC Implementation (parallel with Step 4b)

Read `.flutter-dev/01-architecture.md`, `.flutter-dev/02-state-design.md`, and `.flutter-dev/03-data-design.md`.

Launch this task in parallel with Step 4b using multiple Task tool calls in a single response:

```
Task:
  subagent_type: "flutter-state-expert"
  description: "Implement BLoC classes for $FEATURE"
  prompt: |
    Implement all BLoC classes for this Flutter feature: $FEATURE

    ## Architecture
    [Insert full contents of .flutter-dev/01-architecture.md]

    ## State Management Design
    [Insert full contents of .flutter-dev/02-state-design.md]

    ## Data Layer Design
    [Insert full contents of .flutter-dev/03-data-design.md]

    ## Instructions
    Write the complete, production-ready implementation of every BLoC class
    designed in the state management specification:

    1. **Sealed Event Classes** — Implement every sealed event class exactly as
       specified. Use action-oriented names. Include all typed properties.
       Extend Equatable with correct `props` override.

    2. **Sealed State Classes** — Implement every sealed state class exactly as
       specified. All fields MUST be `final`. Include `props` override for
       Equatable. Add `copyWith` where the design calls for it. Include
       `isProcessing` getters where specified.

    3. **BLoC Classes** — For each BLoC:
       - Accept repository interfaces via constructor (constructor injection only)
       - Register every event handler with its EXPLICIT transformer:
         `on<Event>(handler, transformer: sequential())` — never a bare `on<>()`
       - Use `switch` expression on event type inside logically scoped handlers
         where multiple related events share a transformer
       - Use `emit.forEach` or `onEach` for repository stream subscriptions
       - Never import Flutter — BLoC layer is pure Dart + bloc package
       - No public methods beyond inherited `add()`, `stream`, `state`
       - No `getIt` or service locator calls inside the BLoC

    4. **Error Handling** — Catch domain exceptions from repositories and emit
       appropriate error states. Never catch generic `Exception` without
       rethrowing unexpected errors.

    5. **File Organization** — Follow the package structure from the architecture.
       One BLoC per file, or single-file per BLoC with events/states if the
       design specifies it.

    ## Rules
    - No Cubit — BLoC only
    - Every on<> MUST have an explicit transformer — default is sequential()
    - Sealed classes + Equatable for all events and states
    - Constructor injection only — no getIt inside BLoC
    - No Flutter imports in BLoC layer
    - Never emit the same state object by reference — always create new instances

    Write your complete BLoC implementation as a single markdown document.
```

Save the agent's output to `.flutter-dev/04a-bloc.md`.

Update `state.json`: add `"04a-bloc.md"` to `files_created`, add step "4a" to `completed_steps`, set `last_updated`.

### Step 4b: Data Layer Implementation (parallel with Step 4a)

Read `.flutter-dev/01-architecture.md`, `.flutter-dev/02-state-design.md`, and `.flutter-dev/03-data-design.md`.

Launch this task in parallel with Step 4a:

```
Task:
  subagent_type: "flutter-data-engineer"
  description: "Implement data layer for $FEATURE"
  prompt: |
    Implement the complete data layer for this Flutter feature: $FEATURE

    ## Architecture
    [Insert full contents of .flutter-dev/01-architecture.md]

    ## State Management Design
    [Insert full contents of .flutter-dev/02-state-design.md]

    ## Data Layer Design
    [Insert full contents of .flutter-dev/03-data-design.md]

    ## Instructions
    Write the complete, production-ready implementation of every data layer
    component specified in the data layer design:

    1. **Domain Entities** — Implement pure Dart entity classes with Equatable.
       No JSON annotations, no Flutter imports. Include `props` override and
       `copyWith` methods. These classes live in the domain layer.

    2. **DTOs** — Implement all DTO classes with `@JsonSerializable(fieldRename: FieldRename.snake)`.
       Include `factory fromJson(Map<String, dynamic> json)` and
       `Map<String, dynamic> toJson()`. DTOs live in the data layer only.

    3. **Mappers** — Implement mapper classes or extension methods for converting
       between DTOs and entities. For bidirectional or complex mappings, use
       dedicated mapper classes with constructor-injected dependencies. For
       simple one-way mappings, use extension methods on the DTO.

    4. **Repository Implementations** — Implement every abstract repository
       interface. Mark as `@immutable`. Inject data sources via constructor.
       Map all DTOs to entities before returning. Catch DioException,
       DatabaseException, etc. and rethrow as domain-specific exceptions.
       Never expose data-layer types to callers.

    5. **API Clients** — Implement Retrofit abstract clients with proper
       annotations (@GET, @POST, @PUT, @DELETE, @Query, @Body, @Path).
       Or implement Dio-based clients with proper BaseOptions and interceptors.

    6. **Drift Components (if applicable)** — Implement Drift table definitions,
       DAOs with query methods, database class with migration strategy,
       reactive queries with `watch()`.

    7. **Data Sources** — Implement local and remote data source classes if the
       design calls for them. Handle caching, offline fallback, and sync logic.

    ## Rules
    - No Hive — Drift for SQL, SharedPreferences/SecureStorage for key-value
    - Entities: pure Dart + Equatable, no JSON, no Flutter
    - DTOs: json_serializable with FieldRename.snake
    - Repositories: @immutable, constructor injection, map at boundary
    - Domain exceptions at boundary — never expose DioException
    - Never return DTOs from repository methods — always map to entities

    Write your complete data layer implementation as a single markdown document.
```

Save the agent's output to `.flutter-dev/04b-data.md`.

Update `state.json`: add `"04b-data.md"` to `files_created`, add step "4b" to `completed_steps`, set `last_updated`.

### Step 4c: UI Implementation (depends on Steps 4a + 4b)

Wait for Steps 4a and 4b to complete. Then read `.flutter-dev/01-architecture.md`, `.flutter-dev/02-state-design.md`, `.flutter-dev/03-data-design.md`, `.flutter-dev/04a-bloc.md`, and `.flutter-dev/04b-data.md`.

Use the Task tool:

```
Task:
  subagent_type: "flutter-ui-expert"
  description: "Implement UI layer for $FEATURE"
  prompt: |
    Implement the complete UI layer for this Flutter feature: $FEATURE

    ## Architecture
    [Insert full contents of .flutter-dev/01-architecture.md]

    ## State Management Design
    [Insert full contents of .flutter-dev/02-state-design.md]

    ## Data Layer Design
    [Insert full contents of .flutter-dev/03-data-design.md]

    ## BLoC Implementation
    [Insert full contents of .flutter-dev/04a-bloc.md]

    ## Data Layer Implementation
    [Insert full contents of .flutter-dev/04b-data.md]

    ## Instructions
    Write the complete, production-ready implementation of every screen, widget,
    and UI component for this feature:

    1. **Screen Widgets** — Implement every screen as a StatelessWidget (default).
       Use StatefulWidget ONLY for ephemeral state like TextEditingController,
       AnimationController, or ScrollController. Each screen provides its BLoC
       via BlocProvider at the top.

    2. **BlocBuilder Wiring** — Use `BlocBuilder` with a `switch` expression on
       the sealed state type to render the correct widget per state. Never use
       `if/else` chains on state — use exhaustive `switch`. Add `buildWhen`
       to limit unnecessary rebuilds.

    3. **BlocListener Wiring** — Use `BlocListener` for side effects (navigation,
       snackbars, dialogs). Extract listener callbacks into private methods for
       readability. Add `listenWhen` to filter relevant state transitions.
       Never trigger side effects inside `builder`.

    4. **BlocConsumer** — Use when a widget needs both builder and listener.

    5. **Widget Decomposition** — Extract private widget classes for complex
       subtrees. Keep build methods small and focused. Use `const` constructors
       wherever possible. Apply proper Key strategy for list items.

    6. **Context Extensions** — Define context extensions for repeated
       `context.read<Bloc>().add(Event())` patterns to reduce boilerplate.

    7. **Theming** — Use `Theme.of(context)` for all colors and text styles.
       Never hardcode colors or text styles. Follow Material 3 or the project's
       design system tokens.

    8. **Localization** — Use localization classes for all user-facing strings.
       No hardcoded strings in widgets.

    9. **Navigation** — Wire up go_router routes for the feature's screens.
       Define path parameters and query parameters as needed. Add redirect
       guards if authentication is required.

    10. **Responsive/Adaptive** — Use LayoutBuilder or MediaQuery for responsive
        layouts where needed. Consider tablet and desktop breakpoints if
        the feature targets multiple form factors.

    ## Rules
    - StatelessWidget by default — StatefulWidget only for ephemeral state
    - BlocBuilder with switch expression on sealed states — never if/else
    - BlocListener for side effects only — never in builder
    - Theme.of(context) always — never hardcode colors or text styles
    - Localization classes for all strings — no hardcoded user-facing text
    - Small focused widgets — extract private widget classes

    Write your complete UI implementation as a single markdown document.
```

Save the agent's output to `.flutter-dev/04c-ui.md`.

Update `state.json`: set `current_step` to 5, add `"04c-ui.md"` to `files_created`, add step "4c" to `completed_steps`, set `last_updated`.

### Step 5: Testing (parallel with Step 6; depends on Steps 4a–4c)

Read all prior output files: `.flutter-dev/01-architecture.md` through `.flutter-dev/04c-ui.md`.

Launch this task in parallel with Step 6 using multiple Task tool calls in a single response:

```
Task:
  subagent_type: "flutter-testing-expert"
  description: "Create comprehensive test suite for $FEATURE"
  prompt: |
    Create a comprehensive test suite for this Flutter feature: $FEATURE

    ## Architecture
    [Insert full contents of .flutter-dev/01-architecture.md]

    ## State Management Design
    [Insert full contents of .flutter-dev/02-state-design.md]

    ## Data Layer Design
    [Insert full contents of .flutter-dev/03-data-design.md]

    ## BLoC Implementation
    [Insert full contents of .flutter-dev/04a-bloc.md]

    ## Data Layer Implementation
    [Insert full contents of .flutter-dev/04b-data.md]

    ## UI Implementation
    [Insert full contents of .flutter-dev/04c-ui.md]

    ## Instructions
    Write production-ready tests covering BLoC logic and visual regression:

    1. **BLoC Unit Tests** — Using `bloc_test` package, test every BLoC:
       - Use `blocTest<Bloc, State>()` with `act`, `expect`, `verify`
       - Test every event → state transition path
       - Test transformer behavior (sequential ordering, droppable cancellation,
         restartable restart)
       - Mock repository interfaces with mockito
       - Test error states when repositories throw domain exceptions
       - Test initial state
       - Verify repository method calls with `verify()` and `verifyNoMoreInteractions()`

    2. **Golden File Tests** — For each screen and critical UI component:
       - Define golden files for each sealed state variant
       - Mock BLoC with `MockBloc` from `bloc_test`, seed each sealed state
       - Wrap with `BlocProvider.value` and `MaterialApp` for proper theming
       - Load fonts for deterministic rendering
       - Document `--update-goldens` workflow for CI

    ## Rules
    - Every BLoC event must have at least one blocTest
    - Every sealed state branch must have a golden file
    - Mock repositories at BLoC boundary — never hit real APIs in unit tests
    - Test error paths, not just happy paths

    Write your complete test suite as a single markdown document.
```

Save the agent's output to `.flutter-dev/05-tests.md`.

Update `state.json`: add `"05-tests.md"` to `files_created`, add step 5 to `completed_steps`, set `last_updated`.

### Step 6: Performance Review (parallel with Step 5; depends on Steps 4a–4c)

Read `.flutter-dev/04a-bloc.md`, `.flutter-dev/04b-data.md`, and `.flutter-dev/04c-ui.md`.

Launch this task in parallel with Step 5:

```
Task:
  subagent_type: "flutter-performance-engineer"
  description: "Performance review of $FEATURE implementation"
  prompt: |
    Perform a thorough performance review of this Flutter feature: $FEATURE

    ## BLoC Implementation
    [Insert full contents of .flutter-dev/04a-bloc.md]

    ## Data Layer Implementation
    [Insert full contents of .flutter-dev/04b-data.md]

    ## UI Implementation
    [Insert full contents of .flutter-dev/04c-ui.md]

    ## Instructions
    Review every implementation file for performance issues and optimization
    opportunities:

    1. **Widget Rebuild Analysis** — Identify unnecessary widget rebuilds:
       - BlocBuilder without `buildWhen` where it should have one
       - Large widget trees inside BlocBuilder that could be decomposed
       - Missing `const` constructors on static widgets
       - Incorrect or missing Key usage on list items
       - setState calls that could be narrowed in scope

    2. **RepaintBoundary Assessment** — Identify where RepaintBoundary is needed:
       - Animated widgets without isolation
       - Scrolling lists with complex item rendering
       - CustomPainter widgets missing `repaint` parameter
       - Widgets that change frequently adjacent to static content

    3. **Memory and Resource Issues** — Check for:
       - Streams or controllers not disposed in BLoC `close()`
       - TextEditingController or AnimationController not disposed
       - Large image loading without caching or resizing
       - Unbounded list growth in state

    4. **Data Layer Performance** — Review for:
       - N+1 query patterns in Drift DAOs
       - Missing database indexes on frequently queried columns
       - Unbatched API calls that could be batched
       - Missing pagination on large data sets
       - Excessive JSON parsing on the main isolate

    5. **Animation Performance** — If the feature includes animations:
       - AnimatedBuilder vs setState for animation listeners
       - Paint object reuse (defined as field, not created in paint())
       - Canvas operations that could be batched
       - Tree structure changes during active animations

    6. **Profiling Recommendations** — Suggest specific DevTools profiling
       workflows:
       - Which screens to profile for frame rendering
       - Where to add Timeline events for custom measurements
       - Memory profiling targets
       - Specific widget inspector checks

    ## Deliverables
    1. Performance findings with severity (critical/high/medium/low)
    2. Specific code locations and recommended fixes for each finding
    3. RepaintBoundary placement recommendations
    4. Profiling workflow recommendations
    5. Estimated impact of each optimization

    Write your complete performance review as a single markdown document.
```

Save the agent's output to `.flutter-dev/06-performance.md`.

Update `state.json`: set `current_step` to "checkpoint-2", set `current_phase` to 2, add `"06-performance.md"` to `files_created`, add step 6 to `completed_steps`, set `last_updated`.

---

## PHASE CHECKPOINT 2 — User Approval Required

You MUST stop here and present the implementation phase for review.

Display a summary from `.flutter-dev/04a-bloc.md`, `.flutter-dev/04b-data.md`, `.flutter-dev/04c-ui.md`, `.flutter-dev/05-tests.md`, and `.flutter-dev/06-performance.md` (BLoC classes implemented, data layer components, screens built, test coverage summary, performance findings by severity) and ask:

```
Implementation, testing, and performance review complete. Please review:
- .flutter-dev/04a-bloc.md
- .flutter-dev/04b-data.md
- .flutter-dev/04c-ui.md
- .flutter-dev/05-tests.md
- .flutter-dev/06-performance.md

1. Approve — proceed to platform & deployment
2. Request changes — tell me what to adjust
3. Pause — save progress and stop here
```

Do NOT proceed to Phase 3 until the user selects option 1. If they select option 2, revise the relevant step(s) and re-checkpoint. If option 3, update `state.json` status and stop.

---

## Phase 3: Delivery (Step 7) — Sequential

### Step 7: Platform & Deployment

Read all prior output files: `.flutter-dev/01-architecture.md` through `.flutter-dev/06-performance.md`.

Use the Task tool:

```
Task:
  subagent_type: "flutter-platform-engineer"
  description: "Platform-specific adjustments and deployment setup for $FEATURE"
  prompt: |
    Prepare platform-specific adjustments and deployment configuration for
    this Flutter feature: $FEATURE

    ## Architecture
    [Insert full contents of .flutter-dev/01-architecture.md]

    ## State Management Design
    [Insert full contents of .flutter-dev/02-state-design.md]

    ## Data Layer Design
    [Insert full contents of .flutter-dev/03-data-design.md]

    ## BLoC Implementation
    [Insert full contents of .flutter-dev/04a-bloc.md]

    ## Data Layer Implementation
    [Insert full contents of .flutter-dev/04b-data.md]

    ## UI Implementation
    [Insert full contents of .flutter-dev/04c-ui.md]

    ## Test Suite
    [Insert full contents of .flutter-dev/05-tests.md]

    ## Performance Review
    [Insert full contents of .flutter-dev/06-performance.md]

    ## Instructions
    Review the complete feature implementation and prepare it for multi-platform
    deployment:

    1. **iOS-Specific Adjustments** — Review for iOS platform requirements:
       - Info.plist entries needed (permissions, URL schemes, capabilities)
       - Podfile dependencies and configuration
       - Face ID / Touch ID integration if authentication is involved
       - Haptic feedback opportunities
       - Live Activities or WidgetKit if applicable
       - App Clips consideration
       - App Transport Security configuration if new API endpoints

    2. **Android-Specific Adjustments** — Review for Android platform requirements:
       - AndroidManifest.xml permissions and features
       - Gradle dependencies and configuration
       - BiometricPrompt integration if authentication is involved
       - Material You dynamic theming support
       - Home screen widget opportunities
       - App Links / Deep Links configuration
       - ProGuard/R8 rules for new dependencies

    3. **Web-Specific Adjustments (if applicable)** — Review for web deployment:
       - PWA manifest updates
       - JS interop requirements via dart:js_interop
       - Web-specific rendering considerations
       - SEO metadata if the feature has public-facing pages

    4. **Desktop-Specific Adjustments (if applicable)** — Review for desktop:
       - Window management considerations
       - System tray or menu bar integration
       - File system access requirements
       - Keyboard shortcuts

    5. **Flavor Configuration** — Define per-environment settings:
       - Dev/staging/prod API endpoints for new services
       - --dart-define or --dart-define-from-file entries
       - Per-flavor bundle ID updates if needed
       - Feature flags for gradual rollout

    6. **CI/CD Setup** — Define pipeline additions:
       - New test steps for the feature's test suite
       - Build configuration changes
       - Golden file update workflow
       - Code signing requirements for new capabilities

    7. **Store Readiness** — Prepare for app store submission:
       - New permission descriptions for App Store / Play Console
       - Screenshots or preview updates needed
       - Release notes draft for the feature
       - --obfuscate and --split-debug-info verification

    ## Deliverables
    1. Platform-specific configuration files and adjustments
    2. Flavor/environment configuration additions
    3. CI/CD pipeline updates
    4. Store submission preparation checklist
    5. Platform-specific testing recommendations

    Write your complete platform and deployment plan as a single markdown document.
```

Save the agent's output to `.flutter-dev/07-platform.md`.

Update `state.json`: set `current_step` to "complete", add `"07-platform.md"` to `files_created`, add step 7 to `completed_steps`, set `last_updated`.

---

## Completion

Update `state.json`:

- Set `status` to `"complete"`
- Set `last_updated` to current timestamp

Present the final summary:

```
Flutter feature development complete: $FEATURE

## Files Created
- .flutter-dev/01-architecture.md
- .flutter-dev/02-state-design.md
- .flutter-dev/03-data-design.md
- .flutter-dev/04a-bloc.md
- .flutter-dev/04b-data.md
- .flutter-dev/04c-ui.md
- .flutter-dev/05-tests.md
- .flutter-dev/06-performance.md
- .flutter-dev/07-platform.md

## Implementation Summary

### Phase 1: Discovery
- Architecture: Clean Architecture layer boundaries, package structure, DI plan
- State Management: BLoC classes with sealed events/states, explicit transformers
- Data Layer: Entities, DTOs, repository implementations, API contracts

### Phase 2: Implementation
- BLoC Implementation: All BLoC classes with event handlers and transformers
- Data Layer Implementation: Repositories, API clients, mappers, Drift schemas
- UI Implementation: Screens, widgets, BlocBuilder/Listener wiring, navigation
- Testing: Unit tests, widget tests, golden files, integration test skeletons
- Performance: Rebuild analysis, RepaintBoundary recommendations, profiling plan

### Phase 3: Delivery
- Platform: iOS/Android/Web/Desktop specific adjustments
- Deployment: Flavor config, CI/CD pipeline, store readiness

## Success Criteria
- Architecture reviewed and approved before implementation
- BLoC classes use explicit transformers, sealed states/events with Equatable
- All DTOs mapped to entities at repository boundary — no DTO leakage
- Widget tests cover all sealed state branches with exhaustive switch
- Performance review completed with rebuild analysis and RepaintBoundary recommendations
- Platform-specific adjustments documented for all target platforms

## Next Steps
1. Review all generated code and documentation
2. Apply performance recommendations from .flutter-dev/06-performance.md
3. Run the full test suite: flutter test
4. Update golden files if needed: flutter test --update-goldens
5. Create a pull request with the implementation
6. Deploy using flavor configuration from .flutter-dev/07-platform.md
```
