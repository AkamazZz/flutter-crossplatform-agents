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

- Unit — BLoCs, mappers, domain logic — fast, low–medium confidence
- Widget — individual widgets, state UI — medium speed, medium confidence
- Golden — visual snapshots, pixel diff — medium speed, medium confidence
- Integration — full app, native interactions — slow, high confidence

Write the most tests at the bottom (Unit) and the fewest at the top (Integration).

---

## Read Before Answering

Before generating any test code, read the relevant reference file(s).

- `references/unit-testing.md` — BLoC unit tests, mocks, mappers
- `references/widget-testing.md` — widget rendering, BlocBuilder, finders
- `references/golden-file-testing.md` — visual snapshots, CI golden workflow
- `references/integration-testing.md` — end-to-end, Patrol, device farms

---

## Quick Reference

- **Unit** — BLoC events→states, mappers, domain exceptions — `bloc_test`, `mockito` — every BLoC, mapper, repository
- **Widget** — widget tree per sealed state branch — `testWidgets`, `BlocProvider` — every screen, non-trivial widget
- **Golden** — pixel-accurate snapshot per theme/variant — `matchesGoldenFile` — design system components, screen layouts
- **Integration** — user flows, native dialogs, permissions — `patrol`, `integration_test` — critical journeys, release gates

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
