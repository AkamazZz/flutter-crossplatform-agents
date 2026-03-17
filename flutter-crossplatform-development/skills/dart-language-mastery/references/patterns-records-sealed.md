---
description: >
  Dart 3 language features: pattern matching, records, sealed classes, and exhaustive switch expressions.
---

# Patterns, Records, and Sealed Classes

Dart 3.0 introduced a unified pattern system, records, and sealed classes. Together they enable exhaustive, type-safe modeling that the compiler can verify.

---

## Pattern Matching: Switch Expressions

Switch expressions replace switch statements and if/else chains. They are expressions — they produce a value.

```dart
String describe(Shape shape) => switch (shape) {
  Circle(:final radius) => 'Circle r=$radius',
  Rectangle(:final width, :final height) => 'Rect ${width}x$height',
};
```

**Exhaustiveness** is enforced at compile time when switching over a sealed type. If a case is missing, the compiler errors. This is the primary reason to prefer sealed classes over abstract classes for closed hierarchies.

Switch statements also support patterns:

```dart
switch (event) {
  case LoginPressed(:final email, :final password):
    add(AuthLoginRequested(email: email, password: password));
  case LogoutPressed():
    add(AuthLogoutRequested());
}
```

---

## Destructuring

### Positional Destructuring (records, lists, objects)

```dart
// Record
final (x, y) = (10, 20);

// List
final [first, second, ...rest] = items;

// Object — matches on type and extracts fields
final Point(:x, :y) = point;
```

### Named Destructuring

```dart
case AuthSuccess(:final user):
  print(user.email);
```

The `:final field` shorthand is equivalent to `field: final field` — it binds the field to a variable of the same name.

---

## Records

Records are anonymous, immutable, structurally typed aggregates. They are ideal for returning multiple values without defining a dedicated class.

### Positional Records

```dart
(String, int) nameAndAge() => ('Ada', 36);

final (name, age) = nameAndAge();
```

### Named Records

```dart
({String name, int age}) person() => (name: 'Ada', age: 36);

final (:name, :age) = person();
print('$name is $age');
```

### Records as Type-Safe Tuples

Records carry static type information. Unlike `List<dynamic>` or `Map<String, dynamic>`, a record type `(String, int)` is checked at compile time.

```dart
// Return a result and a status without a wrapper class
(List<User>, bool hasMore) fetchPage() { ... }

final (users, hasMore) = await fetchPage();
```

### Record Equality

Records have structural equality by default — two records are equal if all their fields are equal.

```dart
(1, 'a') == (1, 'a'); // true
```

---

## Sealed Class Hierarchies

A `sealed` class is abstract and closed: all direct subclasses must be in the same library file. The compiler knows the complete set of subtypes, enabling exhaustive switching.

```dart
// states.dart
sealed class AuthState {}

final class AuthInitial extends AuthState {}

final class AuthLoading extends AuthState {}

final class AuthSuccess extends AuthState {
  const AuthSuccess(this.user);
  final User user;
}

final class AuthFailure extends AuthState {
  const AuthFailure(this.message);
  final String message;
}
```

`final` on the subclasses prevents further extension outside the file and keeps the hierarchy flat and explicit.

### Exhaustive Switch

```dart
Widget build(BuildContext context, AuthState state) => switch (state) {
  AuthInitial() => const WelcomeScreen(),
  AuthLoading() => const CircularProgressIndicator(),
  AuthSuccess(:final user) => UserDashboard(user: user),
  AuthFailure(:final message) => ErrorBanner(message: message),
};
```

If `AuthSuccess` is added later and this switch is not updated, the compiler emits an error.

### Flutter: Sealed States in BlocBuilder

```dart
BlocBuilder<AuthBloc, AuthState>(
  builder: (context, state) => switch (state) {
    AuthInitial() => const LoginForm(),
    AuthLoading() => const LoadingSpinner(),
    AuthSuccess(:final user) => HomePage(user: user),
    AuthFailure(:final message) => ErrorView(message: message),
  },
)
```

---

## Guard Clauses

A `when` clause adds a boolean condition to a pattern arm. The pattern must match structurally first, then the guard is evaluated.

```dart
switch (event) {
  case PageScrolled(:final offset) when offset > 200:
    _showBackToTop();
  case PageScrolled():
    _hideBackToTop();
}
```

Guards do not contribute to exhaustiveness analysis. If all arms with a guard fail, Dart falls through to the next arm (or throws a StateError if no arm matches).

---

## Logical Patterns

Combine patterns with `||` (logical-or) and `&&` (logical-and).

```dart
switch (value) {
  case int x when x < 0 || x > 100:
    throw RangeError(value);
  case int x:
    process(x);
}
```

---

## Relational Patterns

Match a value against a constant using `<`, `<=`, `>`, `>=`, `==`, `!=`.

```dart
String classify(int score) => switch (score) {
  >= 90 => 'A',
  >= 80 => 'B',
  >= 70 => 'C',
  _ => 'F',
};
```

---

## Null-Check and Null-Assert Patterns

```dart
// null-check: matches only if value is non-null, binds non-nullable
case String? s?:   // matches if s != null
  print(s.length);

// null-assert: asserts non-null, throws if null
case String? s!:
  print(s.length);
```

---

## Common Mistakes

**Using abstract classes when sealed is needed.** An abstract class allows external subclassing, so the compiler cannot guarantee exhaustiveness. Use `sealed` for closed state hierarchies.

**Forgetting that guards break exhaustiveness.** A switch that has guards on every arm is not exhaustive from the compiler's perspective. Include a catch-all arm or a guard-free arm if needed.

**Using sealed for open extension points.** If third-party code legitimately needs to subclass, use abstract instead. Sealed is only for closed, internally-owned hierarchies.
