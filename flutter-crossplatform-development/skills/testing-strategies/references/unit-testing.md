---
description: >
  Unit testing for BLoC with bloc_test, mockito for repositories, and TDD red-green-refactor workflow.
---

# Unit Testing in Flutter

Unit tests verify pure logic: BLoCs, Cubits, mappers, domain objects, and use cases. No Flutter framework is required; tests run on the Dart VM and finish in milliseconds.

---

## bloc_test — `blocTest()` Pattern

The `blocTest` function from the `bloc_test` package follows a `build / act / expect` structure. All three phases are optional but should be used deliberately.

```dart
blocTest<CounterBloc, CounterState>(
  'description of what this test verifies',
  build: () => CounterBloc(),          // (1) build — instantiate the bloc
  act:   (bloc) => bloc.add(Increment()), // (2) act — dispatch one or more events
  expect: () => [                      // (3) expect — ordered list of emitted states
    const CounterState(count: 1),
  ],
);
```

### Full Example

```dart
import 'package:bloc_test/bloc_test.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:mocktail/mocktail.dart';

import 'package:my_app/counter/bloc/counter_bloc.dart';
import 'package:my_app/counter/bloc/counter_event.dart';
import 'package:my_app/counter/bloc/counter_state.dart';

void main() {
  group('CounterBloc', () {
    late CounterBloc bloc;

    setUp(() => bloc = CounterBloc());
    tearDown(() => bloc.close());

    test('initial state is CounterInitial with count 0', () {
      expect(bloc.state, const CounterState(count: 0));
    });

    blocTest<CounterBloc, CounterState>(
      'emits [CounterState(1)] when Increment is added',
      build: () => CounterBloc(),
      act: (b) => b.add(Increment()),
      expect: () => [const CounterState(count: 1)],
    );

    blocTest<CounterBloc, CounterState>(
      'emits [CounterState(-1)] when Decrement is added',
      build: () => CounterBloc(),
      act: (b) => b.add(Decrement()),
      expect: () => [const CounterState(count: -1)],
    );

    blocTest<CounterBloc, CounterState>(
      'emits [1, 2, 3] for three consecutive Increments',
      build: () => CounterBloc(),
      act: (b) {
        b.add(Increment());
        b.add(Increment());
        b.add(Increment());
      },
      expect: () => [
        const CounterState(count: 1),
        const CounterState(count: 2),
        const CounterState(count: 3),
      ],
    );
  });
}
```

---

## Testing Sealed State Sequences

Sealed classes model exhaustive state machines. Every variant must appear in at least one test. Assert on exact emit order to catch regressions in transition logic.

```dart
// State definition
sealed class SearchState {}
final class SearchInitial  extends SearchState {}
final class SearchLoading  extends SearchState {}
final class SearchLoaded   extends SearchState { final List<String> results; ... }
final class SearchError    extends SearchState { final String message; ... }

// Tests
blocTest<SearchBloc, SearchState>(
  'emits [Loading, Loaded] on successful search',
  build: () => SearchBloc(repo: mockRepo),
  setUp: () => when(() => mockRepo.search('q')).thenAnswer((_) async => ['result']),
  act: (b) => b.add(SearchQueryChanged('q')),
  expect: () => [
    isA<SearchLoading>(),
    isA<SearchLoaded>().having((s) => s.results, 'results', ['result']),
  ],
);

blocTest<SearchBloc, SearchState>(
  'emits [Loading, Error] when repository throws',
  build: () => SearchBloc(repo: mockRepo),
  setUp: () => when(() => mockRepo.search(any())).thenThrow(NetworkException()),
  act: (b) => b.add(SearchQueryChanged('q')),
  expect: () => [
    isA<SearchLoading>(),
    isA<SearchError>().having((s) => s.message, 'message', contains('network')),
  ],
);
```

---

## Testing Event Transformers

`bloc` supports transformer composition. Tests must verify the transformer's contract, not just the state output.

### Droppable — drops events while one is in flight

```dart
blocTest<SearchBloc, SearchState>(
  'droppable: only first event is processed when two arrive concurrently',
  build: () => SearchBloc(repo: mockRepo), // transformer: droppable()
  act: (b) {
    b.add(SearchQueryChanged('a'));
    b.add(SearchQueryChanged('b')); // dropped
  },
  // Only one Loading+Loaded pair, not two
  expect: () => [isA<SearchLoading>(), isA<SearchLoaded>()],
  verify: (_) => verify(() => mockRepo.search(any())).called(1),
);
```

### Restartable — cancels in-flight event when a new one arrives

```dart
blocTest<SearchBloc, SearchState>(
  'restartable: second event cancels first',
  build: () => SearchBloc(repo: mockRepo), // transformer: restartable()
  act: (b) async {
    b.add(SearchQueryChanged('a'));
    await Future<void>.delayed(const Duration(milliseconds: 10));
    b.add(SearchQueryChanged('b')); // restarts, cancels 'a' fetch
  },
  expect: () => [
    isA<SearchLoading>(), // from 'a'
    isA<SearchLoading>(), // from 'b'
    isA<SearchLoaded>(),  // result of 'b'
  ],
);
```

### Sequential — queues events in order

```dart
blocTest<QueueBloc, QueueState>(
  'sequential: processes events one by one in order',
  build: () => QueueBloc(repo: mockRepo), // transformer: sequential()
  act: (b) {
    b.add(ProcessItem('first'));
    b.add(ProcessItem('second'));
  },
  expect: () => [
    isA<QueueProcessing>().having((s) => s.current, 'current', 'first'),
    isA<QueueDone>(),
    isA<QueueProcessing>().having((s) => s.current, 'current', 'second'),
    isA<QueueDone>(),
  ],
);
```

---

## Mockito / Mocktail — Generating Mocks

Use `@GenerateMocks` (mockito) or class-level `extends Mock` (mocktail). Prefer mocktail for null-safe projects to avoid build_runner overhead when annotations are not needed.

### mockito with code generation

```dart
// test/mocks.dart
import 'package:mockito/annotations.dart';
import 'package:my_app/data/repositories/user_repository.dart';

@GenerateMocks([UserRepository])
void main() {}
```

Run `dart run build_runner build` to generate `mocks.mocks.dart`, then:

```dart
import 'mocks.mocks.dart';

final mockRepo = MockUserRepository();
when(mockRepo.getUser(id: anyNamed('id')))
    .thenAnswer((_) async => userFixture);
```

### mocktail (no code generation)

```dart
class MockUserRepository extends Mock implements UserRepository {}

setUp(() {
  mockRepo = MockUserRepository();
  registerFallbackValue(const UserId('fallback'));
});

when(() => mockRepo.getUser(id: any(named: 'id')))
    .thenAnswer((_) async => userFixture);
```

---

## Testing Domain Exceptions

Domain exceptions carry meaning. Assert on type and message independently.

```dart
blocTest<AuthBloc, AuthState>(
  'emits AuthError with InvalidCredentials on 401',
  build: () => AuthBloc(repo: mockRepo),
  setUp: () => when(() => mockRepo.signIn(any(), any()))
      .thenThrow(const InvalidCredentialsException()),
  act: (b) => b.add(SignInRequested(email: 'x@y.com', password: 'wrong')),
  expect: () => [
    isA<AuthLoading>(),
    isA<AuthError>().having(
      (s) => s.failure,
      'failure',
      isA<InvalidCredentialsFailure>(),
    ),
  ],
);
```

---

## Testing Mappers (DTO → Entity Roundtrip)

Mappers must be pure functions. Test both directions and edge cases (null fields, empty lists).

```dart
void main() {
  group('UserMapper', () {
    const mapper = UserMapper();

    test('fromDto converts all fields correctly', () {
      final dto = UserDto(id: '1', name: 'Alice', avatarUrl: null);
      final entity = mapper.fromDto(dto);

      expect(entity.id.value, '1');
      expect(entity.name, 'Alice');
      expect(entity.avatarUrl, isNull);
    });

    test('toDto converts entity back to dto (roundtrip)', () {
      const entity = User(id: UserId('1'), name: 'Alice', avatarUrl: null);
      final dto = mapper.toDto(entity);
      final restored = mapper.fromDto(dto);

      expect(restored, entity);
    });

    test('fromDto handles empty roles list', () {
      final dto = UserDto(id: '1', name: 'Bob', roles: []);
      expect(mapper.fromDto(dto).roles, isEmpty);
    });
  });
}
```

---

## Complete BLoC Test File Example

```dart
// test/features/auth/bloc/auth_bloc_test.dart

import 'package:bloc_test/bloc_test.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:mocktail/mocktail.dart';

import 'package:my_app/features/auth/bloc/auth_bloc.dart';
import 'package:my_app/features/auth/bloc/auth_event.dart';
import 'package:my_app/features/auth/bloc/auth_state.dart';
import 'package:my_app/features/auth/domain/repositories/auth_repository.dart';
import 'package:my_app/features/auth/domain/exceptions/auth_exception.dart';
import 'package:my_app/features/auth/domain/entities/user.dart';

class MockAuthRepository extends Mock implements AuthRepository {}

void main() {
  late MockAuthRepository mockRepo;

  setUp(() {
    mockRepo = MockAuthRepository();
    registerFallbackValue(const SignInParams(email: '', password: ''));
  });

  group('AuthBloc', () {
    group('SignInRequested', () {
      const params = SignInParams(email: 'user@test.com', password: 'secret');
      const user  = User(id: UserId('u1'), email: 'user@test.com');

      blocTest<AuthBloc, AuthState>(
        'emits [AuthLoading, AuthAuthenticated] on success',
        build: () => AuthBloc(repo: mockRepo),
        setUp: () => when(() => mockRepo.signIn(params))
            .thenAnswer((_) async => user),
        act: (b) => b.add(SignInRequested(params: params)),
        expect: () => [
          isA<AuthLoading>(),
          isA<AuthAuthenticated>().having((s) => s.user, 'user', user),
        ],
        verify: (_) => verify(() => mockRepo.signIn(params)).called(1),
      );

      blocTest<AuthBloc, AuthState>(
        'emits [AuthLoading, AuthError] on InvalidCredentialsException',
        build: () => AuthBloc(repo: mockRepo),
        setUp: () => when(() => mockRepo.signIn(any()))
            .thenThrow(const InvalidCredentialsException()),
        act: (b) => b.add(SignInRequested(params: params)),
        expect: () => [
          isA<AuthLoading>(),
          isA<AuthError>().having(
            (s) => s.failure,
            'failure',
            isA<InvalidCredentialsFailure>(),
          ),
        ],
      );

      blocTest<AuthBloc, AuthState>(
        'emits [AuthLoading, AuthError] on unexpected exception',
        build: () => AuthBloc(repo: mockRepo),
        setUp: () => when(() => mockRepo.signIn(any()))
            .thenThrow(Exception('boom')),
        act: (b) => b.add(SignInRequested(params: params)),
        expect: () => [
          isA<AuthLoading>(),
          isA<AuthError>().having(
            (s) => s.failure,
            'failure',
            isA<UnexpectedFailure>(),
          ),
        ],
      );
    });

    group('SignOutRequested', () {
      blocTest<AuthBloc, AuthState>(
        'emits [AuthUnauthenticated] on sign out',
        build: () => AuthBloc(repo: mockRepo),
        seed: () => const AuthAuthenticated(
          user: User(id: UserId('u1'), email: 'user@test.com'),
        ),
        setUp: () => when(() => mockRepo.signOut()).thenAnswer((_) async {}),
        act: (b) => b.add(SignOutRequested()),
        expect: () => [isA<AuthUnauthenticated>()],
      );
    });
  });
}
```
