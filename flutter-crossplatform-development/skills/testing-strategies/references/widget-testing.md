---
description: >
  Widget testing with testWidgets, sealed state matching, finder patterns, and pump strategies.
---

# Widget Testing in Flutter

Widget tests verify that widgets render correctly and respond to user interactions. They run on a headless Flutter engine — faster than integration tests but slower than unit tests.

---

## Basic `testWidgets` Setup

```dart
import 'package:flutter/material.dart';
import 'package:flutter_test/flutter_test.dart';

void main() {
  testWidgets('CounterPage shows initial count of 0', (tester) async {
    await tester.pumpWidget(
      const MaterialApp(home: CounterPage()),
    );

    expect(find.text('0'), findsOneWidget);
  });
}
```

`tester.pumpWidget` builds the widget tree and triggers the first frame. Always wrap the root widget in `MaterialApp` (or the app's custom wrapper) to provide `Navigator`, `Theme`, `Directionality`, and `MediaQuery`.

---

## Wrapping with BlocProvider and MaterialApp

Widgets that depend on a BLoC must receive it through a `BlocProvider`. Provide a pre-seeded or mock BLoC to keep tests deterministic.

```dart
import 'package:bloc_test/bloc_test.dart';
import 'package:flutter_bloc/flutter_bloc.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:mocktail/mocktail.dart';

class MockAuthBloc extends MockBloc<AuthEvent, AuthState> implements AuthBloc {}

Widget buildSubject({required AuthBloc bloc}) {
  return MaterialApp(
    theme: AppTheme.light(), // use your app's theme for accurate rendering
    home: BlocProvider<AuthBloc>.value(
      value: bloc,
      child: const LoginPage(),
    ),
  );
}

void main() {
  late MockAuthBloc bloc;

  setUp(() => bloc = MockAuthBloc());
  tearDown(() => bloc.close());

  testWidgets('renders email and password fields', (tester) async {
    when(() => bloc.state).thenReturn(const AuthInitial());
    await tester.pumpWidget(buildSubject(bloc: bloc));

    expect(find.byType(EmailField), findsOneWidget);
    expect(find.byType(PasswordField), findsOneWidget);
  });
}
```

Use `MockBloc` (from `bloc_test`) to stub `state` and `stream` without implementing the full BLoC.

---

## Finders

- `find.byType(MyWidget)` — assert a widget class is present
- `find.text('Hello')` — assert exact text appears
- `find.byKey(const Key('id'))` — assert a widget with a semantic key is present
- `find.byIcon(Icons.check)` — assert an icon is present
- `find.ancestor(...)` — scope a finder within a subtree
- `find.descendant(...)` — find a widget inside a specific ancestor
- `find.bySemanticsLabel('...')` — accessibility-oriented assertion

```dart
// Multiple matches
expect(find.byType(ListTile), findsNWidgets(3));

// No match
expect(find.text('Error'), findsNothing);

// Exactly one
expect(find.byKey(const Key('submit-button')), findsOneWidget);
```

---

## Actions — Tapping, Entering Text, Scrolling

### Tapping

```dart
await tester.tap(find.byType(ElevatedButton));
await tester.pump(); // rebuild after tap
```

### Entering text

```dart
await tester.enterText(find.byType(TextField), 'user@example.com');
await tester.pump();
```

### Scrolling

```dart
await tester.drag(find.byType(ListView), const Offset(0, -300));
await tester.pumpAndSettle();
```

### Long press

```dart
await tester.longPress(find.byKey(const Key('card-item')));
await tester.pump();
```

---

## Verifying BlocBuilder Renders Correct Widget per Sealed State Branch

Every sealed state branch must produce a distinct, assertable widget tree.

```dart
group('SearchPage sealed state rendering', () {
  late MockSearchBloc bloc;

  setUp(() => bloc = MockSearchBloc());
  tearDown(() => bloc.close());

  testWidgets('renders SearchInitialView for SearchInitial', (tester) async {
    when(() => bloc.state).thenReturn(const SearchInitial());
    await tester.pumpWidget(buildSubject(bloc: bloc));

    expect(find.byType(SearchInitialView), findsOneWidget);
    expect(find.byType(LoadingIndicator), findsNothing);
  });

  testWidgets('renders LoadingIndicator for SearchLoading', (tester) async {
    when(() => bloc.state).thenReturn(const SearchLoading());
    await tester.pumpWidget(buildSubject(bloc: bloc));

    expect(find.byType(LoadingIndicator), findsOneWidget);
  });

  testWidgets('renders SearchResultsList for SearchLoaded', (tester) async {
    when(() => bloc.state).thenReturn(SearchLoaded(results: ['a', 'b']));
    await tester.pumpWidget(buildSubject(bloc: bloc));

    expect(find.byType(SearchResultsList), findsOneWidget);
    expect(find.text('a'), findsOneWidget);
    expect(find.text('b'), findsOneWidget);
  });

  testWidgets('renders ErrorView for SearchError', (tester) async {
    when(() => bloc.state).thenReturn(const SearchError(message: 'Network error'));
    await tester.pumpWidget(buildSubject(bloc: bloc));

    expect(find.byType(ErrorView), findsOneWidget);
    expect(find.text('Network error'), findsOneWidget);
  });
});
```

---

## `pump()` vs `pumpAndSettle()`

- `pump()` — triggers exactly one frame; use after sync state changes or when controlling timing
- `pump(Duration(…))` — advances the clock by the given duration; use after async delays, timers, or animations with known durations
- `pumpAndSettle()` — repeatedly pumps until no pending frames remain (max 100); use after animations that must fully complete

`pumpAndSettle` will time out if a looping animation or an unresolved `Future` keeps scheduling frames. Prefer explicit `pump` when possible.

```dart
// Explicit duration pump for a debounce
await tester.pump(const Duration(milliseconds: 300));
expect(find.byType(SearchResultsList), findsOneWidget);

// pumpAndSettle for route transition animation
await tester.tap(find.text('Go to Profile'));
await tester.pumpAndSettle();
expect(find.byType(ProfilePage), findsOneWidget);
```

---

## Testing BlocListener Side Effects

`BlocListener` triggers callbacks (navigation, snackbars, dialogs) without rebuilding. Verify side effects through the widget tree, not the BLoC.

```dart
testWidgets('shows SnackBar when AuthError is emitted', (tester) async {
  final controller = StreamController<AuthState>();
  when(() => bloc.stream).thenAnswer((_) => controller.stream);
  when(() => bloc.state).thenReturn(const AuthInitial());

  await tester.pumpWidget(buildSubject(bloc: bloc));

  controller.add(const AuthError(failure: UnexpectedFailure()));
  await tester.pump(); // deliver the stream event
  await tester.pump(const Duration(milliseconds: 100)); // allow listener to react

  expect(find.byType(SnackBar), findsOneWidget);
  expect(find.text('Something went wrong'), findsOneWidget);

  await controller.close();
});

testWidgets('navigates to HomeScreen when AuthAuthenticated is emitted',
    (tester) async {
  final controller = StreamController<AuthState>();
  when(() => bloc.stream).thenAnswer((_) => controller.stream);
  when(() => bloc.state).thenReturn(const AuthInitial());

  await tester.pumpWidget(buildSubject(bloc: bloc));

  controller.add(AuthAuthenticated(user: userFixture));
  await tester.pumpAndSettle();

  expect(find.byType(HomeScreen), findsOneWidget);

  await controller.close();
});
```

---

## Complete Widget Test Example

```dart
// test/features/auth/view/login_page_test.dart

import 'dart:async';

import 'package:bloc_test/bloc_test.dart';
import 'package:flutter/material.dart';
import 'package:flutter_bloc/flutter_bloc.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:mocktail/mocktail.dart';

import 'package:my_app/features/auth/bloc/auth_bloc.dart';
import 'package:my_app/features/auth/bloc/auth_event.dart';
import 'package:my_app/features/auth/bloc/auth_state.dart';
import 'package:my_app/features/auth/view/login_page.dart';
import 'package:my_app/features/home/view/home_screen.dart';
import 'package:my_app/theme/app_theme.dart';

class MockAuthBloc extends MockBloc<AuthEvent, AuthState> implements AuthBloc {}

void main() {
  late MockAuthBloc bloc;

  setUp(() => bloc = MockAuthBloc());
  tearDown(() => bloc.close());

  Widget buildSubject() => MaterialApp(
        theme: AppTheme.light(),
        home: BlocProvider<AuthBloc>.value(
          value: bloc,
          child: const LoginPage(),
        ),
        routes: {'/home': (_) => const HomeScreen()},
      );

  group('LoginPage', () {
    testWidgets('renders email field, password field, and submit button',
        (tester) async {
      when(() => bloc.state).thenReturn(const AuthInitial());
      await tester.pumpWidget(buildSubject());

      expect(find.byKey(const Key('email-field')), findsOneWidget);
      expect(find.byKey(const Key('password-field')), findsOneWidget);
      expect(find.byKey(const Key('submit-button')), findsOneWidget);
    });

    testWidgets('adds SignInRequested when submit button is tapped',
        (tester) async {
      when(() => bloc.state).thenReturn(const AuthInitial());
      await tester.pumpWidget(buildSubject());

      await tester.enterText(
          find.byKey(const Key('email-field')), 'user@test.com');
      await tester.enterText(
          find.byKey(const Key('password-field')), 'password');
      await tester.tap(find.byKey(const Key('submit-button')));
      await tester.pump();

      verify(() => bloc.add(
            SignInRequested(
              params: const SignInParams(
                email: 'user@test.com',
                password: 'password',
              ),
            ),
          )).called(1);
    });

    testWidgets('shows CircularProgressIndicator for AuthLoading state',
        (tester) async {
      when(() => bloc.state).thenReturn(const AuthLoading());
      await tester.pumpWidget(buildSubject());

      expect(find.byType(CircularProgressIndicator), findsOneWidget);
      expect(find.byKey(const Key('submit-button')), findsNothing);
    });

    testWidgets('shows error message for AuthError state', (tester) async {
      when(() => bloc.state).thenReturn(
        const AuthError(failure: InvalidCredentialsFailure()),
      );
      await tester.pumpWidget(buildSubject());

      expect(find.text('Invalid credentials'), findsOneWidget);
    });

    testWidgets('navigates to HomeScreen on AuthAuthenticated', (tester) async {
      final controller = StreamController<AuthState>();
      when(() => bloc.stream).thenAnswer((_) => controller.stream);
      when(() => bloc.state).thenReturn(const AuthInitial());

      await tester.pumpWidget(buildSubject());

      controller.add(AuthAuthenticated(user: userFixture));
      await tester.pumpAndSettle();

      expect(find.byType(HomeScreen), findsOneWidget);

      await controller.close();
    });
  });
}
```
