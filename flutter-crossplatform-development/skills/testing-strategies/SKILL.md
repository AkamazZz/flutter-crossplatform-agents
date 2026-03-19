---
name: testing-strategies
description: >
  Flutter testing with unit tests, widget tests, golden files, and integration tests. Use when
  writing tests, setting up TDD, debugging test failures, or configuring test infrastructure.
  Trigger on: tests, TDD, unit testing, widget testing, golden files, integration testing,
  Patrol, bloc_test, mockito, coverage.
---

# Flutter Testing Strategies

## Testing Pyramid

```
           /\
          /  \
         / 4  \   Integration (Patrol / integration_test)
        /------\
       /   3    \  Golden File (visual regression)
      /----------\
     /     2      \ Widget Tests (testWidgets)
    /--------------\
   /       1        \ Unit Tests (bloc_test, mockito)
  /------------------\
```

| Layer       | Scope                          | Speed    | Confidence |
|-------------|--------------------------------|----------|------------|
| Unit        | BLoCs, mappers, domain logic   | Fast     | Low–Med    |
| Widget      | Individual widgets, state UI   | Medium   | Medium     |
| Golden      | Visual snapshots, pixel diff   | Medium   | Medium     |
| Integration | Full app, native interactions  | Slow     | High       |

Write the most tests at the bottom (Unit) and the fewest at the top (Integration).

---

## Read Before Answering

Before generating any test code, read the relevant reference file(s).

| Topic                                | Reference File                                          |
|--------------------------------------|---------------------------------------------------------|
| BLoC/Cubit unit tests, mocks, mappers | `references/unit-testing.md`                           |
| Widget rendering, BlocBuilder, finders | `references/widget-testing.md`                        |
| Visual snapshots, CI golden workflow  | `references/golden-file-testing.md`                    |
| End-to-end, Patrol, device farms      | `references/integration-testing.md`                    |

---

## Quick Reference

| Test Type   | What to Test                                   | Tools                          | When                                      |
|-------------|------------------------------------------------|--------------------------------|-------------------------------------------|
| Unit        | BLoC events→states, mappers, domain exceptions | `bloc_test`, `mockito`         | Every BLoC, mapper, repository, use case  |
| Widget      | Widget tree per sealed state branch            | `testWidgets`, `BlocProvider`  | Every screen, every non-trivial widget    |
| Golden      | Pixel-accurate snapshot per theme/variant      | `matchesGoldenFile`            | Design system components, screen layouts  |
| Integration | User flows, native dialogs, permissions        | `patrol`, `integration_test`   | Critical user journeys, release gates     |

---

## Anti-Patterns

### Testing implementation details
Do not assert on internal private variables or call private methods. Test observable outputs: emitted states, rendered widgets, navigation side effects.

```dart
// BAD — testing internal state
expect(bloc.internalCache, isNotEmpty);

// GOOD — testing emitted states
blocTest<SearchBloc, SearchState>(
  'emits loaded state with results',
  build: () => SearchBloc(repo),
  act: (b) => b.add(SearchQueryChanged('flutter')),
  expect: () => [isA<SearchLoaded>()],
);
```

### Mocking too much
If a test requires mocking 5+ collaborators, the production class likely violates Single Responsibility. Refactor first; test the smaller units independently.

### Not updating golden files in CI
Golden files that are never regenerated in CI drift from reality and produce false failures. Establish a dedicated `--update-goldens` step that commits the diff on approved PRs.

### Skipping sealed state branches
Every branch of a sealed state class must be exercised. Missing branches hide runtime crashes when unhandled states reach the UI.

### Using `pumpAndSettle` unconditionally
`pumpAndSettle` loops until no more frames are scheduled. Animations or timers that never settle will time out the test. Prefer explicit `pump(Duration(...))` when timing is relevant.
