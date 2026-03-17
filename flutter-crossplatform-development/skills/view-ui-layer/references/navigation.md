---
description: Navigation patterns — go_router setup, GoRoute/ShellRoute, path/query params, deep linking, auth guards, Navigator 2.0 fundamentals, BLoC-driven redirects.
---

# Navigation

## 1. go_router Setup

The project uses `go_router` as the sole navigation API. All routes are declared
in a single `GoRouter` configuration object, typically in
`lib/core/router/app_router.dart`.

```dart
// lib/core/router/app_router.dart
import 'package:go_router/go_router.dart';

final appRouter = GoRouter(
  initialLocation: '/home',
  debugLogDiagnostics: true,
  routes: [
    GoRoute(
      path: '/login',
      name: AppRoute.login.name,
      builder: (context, state) => const LoginScreen(),
    ),
    ShellRoute(
      builder: (context, state, child) => AppShell(child: child),
      routes: [
        GoRoute(
          path: '/home',
          name: AppRoute.home.name,
          builder: (context, state) => const HomeScreen(),
        ),
        GoRoute(
          path: '/products',
          name: AppRoute.products.name,
          builder: (context, state) => const ProductListScreen(),
          routes: [
            GoRoute(
              path: ':productId',
              name: AppRoute.productDetail.name,
              builder: (context, state) {
                final productId = state.pathParameters['productId']!;
                return ProductDetailScreen(productId: productId);
              },
            ),
          ],
        ),
        GoRoute(
          path: '/profile',
          name: AppRoute.profile.name,
          builder: (context, state) => const ProfileScreen(),
        ),
      ],
    ),
  ],
  redirect: _authGuard,
);
```

Provide `appRouter` to `MaterialApp.router`:

```dart
MaterialApp.router(
  routerConfig: appRouter,
  theme: AppTheme.light,
  darkTheme: AppTheme.dark,
)
```

Use an enum for route names to avoid string typos:

```dart
enum AppRoute {
  login,
  home,
  products,
  productDetail,
  profile,
}
```

---

## 2. GoRoute and ShellRoute

### GoRoute

Declares a single routable destination. Children represent nested paths.

```dart
GoRoute(
  path: '/orders',
  builder: (context, state) => const OrderListScreen(),
  routes: [
    GoRoute(
      path: ':orderId',            // becomes /orders/:orderId
      builder: (context, state) {
        final orderId = state.pathParameters['orderId']!;
        return OrderDetailScreen(orderId: orderId);
      },
    ),
  ],
)
```

### ShellRoute

Wraps child routes in a persistent shell widget (e.g. bottom nav bar, navigation rail).
The shell widget receives the `child` argument — the currently active sub-route — and
renders it inside a shared scaffold.

```dart
ShellRoute(
  builder: (context, state, child) {
    return AppShell(
      currentLocation: state.matchedLocation,
      child: child,
    );
  },
  routes: [
    GoRoute(path: '/home',    builder: (_, __) => const HomeScreen()),
    GoRoute(path: '/search',  builder: (_, __) => const SearchScreen()),
    GoRoute(path: '/profile', builder: (_, __) => const ProfileScreen()),
  ],
)
```

---

## 3. Path Parameters and Query Parameters

### Path parameters

Declared with `:name` in the path string. Retrieved via `state.pathParameters`.

```dart
GoRoute(
  path: '/articles/:slug',
  builder: (context, state) {
    final slug = state.pathParameters['slug']!;
    return ArticleScreen(slug: slug);
  },
)
```

Navigate with named route to avoid constructing paths manually:

```dart
context.goNamed(
  AppRoute.articleDetail.name,
  pathParameters: {'slug': article.slug},
);
```

### Query parameters

Retrieved via `state.uri.queryParameters`:

```dart
GoRoute(
  path: '/search',
  builder: (context, state) {
    final query = state.uri.queryParameters['q'] ?? '';
    final page  = int.tryParse(state.uri.queryParameters['page'] ?? '1') ?? 1;
    return SearchScreen(query: query, page: page);
  },
)
```

Navigate:

```dart
context.go('/search?q=flutter&page=2');
// or
context.goNamed(
  AppRoute.search.name,
  queryParameters: {'q': 'flutter', 'page': '2'},
);
```

---

## 4. Deep Linking

### iOS — Universal Links

Add the associated domains entitlement in `ios/Runner/Runner.entitlements`:

```xml
<key>com.apple.developer.associated-domains</key>
<array>
  <string>applinks:example.com</string>
</array>
```

Host `apple-app-site-association` at `https://example.com/.well-known/apple-app-site-association`:

```json
{
  "applinks": {
    "details": [
      {
        "appIDs": ["TEAMID.com.example.app"],
        "components": [
          { "/": "/products/*" },
          { "/": "/articles/*" }
        ]
      }
    ]
  }
}
```

### Android — App Links

In `android/app/src/main/AndroidManifest.xml`, add an intent filter with
`android:autoVerify="true"`:

```xml
<intent-filter android:autoVerify="true">
  <action android:name="android.intent.action.VIEW" />
  <category android:name="android.intent.category.DEFAULT" />
  <category android:name="android.intent.category.BROWSABLE" />
  <data android:scheme="https" android:host="example.com" />
</intent-filter>
```

Host `assetlinks.json` at `https://example.com/.well-known/assetlinks.json`.

### go_router deep link handling

`go_router` handles incoming links automatically when `GoRouter` is the `routerConfig`.
No additional `onGenerateRoute` wiring is needed.

---

## 5. Redirect Guards for Authentication

Centralise auth guards in the `redirect` callback of `GoRouter`. Read auth state from
a `Notifier` (or `ChangeNotifier`) passed via `refreshListenable`.

```dart
// lib/core/router/app_router.dart
GoRouter(
  refreshListenable: authNotifier,   // triggers re-evaluation on auth change
  redirect: (context, state) {
    final isAuthenticated = authNotifier.isAuthenticated;
    final isOnLogin = state.matchedLocation == '/login';

    if (!isAuthenticated && !isOnLogin) return '/login';
    if (isAuthenticated && isOnLogin)  return '/home';
    return null;  // no redirect
  },
  routes: [...],
)
```

`AuthNotifier` wraps the `AuthBloc` stream and implements `Listenable`:

```dart
class AuthNotifier extends ChangeNotifier {
  AuthNotifier(AuthBloc authBloc) {
    _subscription = authBloc.stream.listen((state) {
      _isAuthenticated = state is AuthStateAuthenticated;
      notifyListeners();
    });
  }

  late final StreamSubscription<AuthState> _subscription;
  bool _isAuthenticated = false;

  bool get isAuthenticated => _isAuthenticated;

  @override
  void dispose() {
    _subscription.cancel();
    super.dispose();
  }
}
```

---

## 6. Navigator 2.0 Fundamentals

`go_router` is built on Navigator 2.0. Understanding the underlying model helps when
debugging or extending routing behaviour:

- **`RouteInformationParser`** — parses a URL string into a structured `RouteInformation`.
- **`RouterDelegate`** — translates `RouteInformation` into a `Navigator` widget tree.
- **`RouteInformationProvider`** — supplies URL updates from the platform (browser bar,
  deep links).

`go_router` provides concrete implementations of all three, which is why direct usage of
`Navigator` 2.0 APIs is rarely needed in application code.

---

## 7. Navigation with BLoC — Auth State Redirects

Pattern: listen to `AuthBloc` in a top-level `BlocListener` and call `context.go()`.

```dart
// In the root widget, above the MaterialApp
BlocListener<AuthBloc, AuthState>(
  listener: (context, state) {
    switch (state) {
      case AuthStateUnauthenticated():
        context.go('/login');
      case AuthStateAuthenticated():
        context.go('/home');
      default:
        break;
    }
  },
  child: MaterialApp.router(routerConfig: appRouter),
)
```

Alternatively, use `AuthNotifier` + `refreshListenable` (shown in section 5) which keeps
routing logic inside the router configuration and avoids `BlocListener` at the app root.

Both approaches are valid; prefer `refreshListenable` when all auth routing is
declarative, and `BlocListener` when specific states require imperative actions
(e.g. showing a logout dialog before redirecting).

---

## 8. Anti-Patterns

1. **`Navigator.push` / `Navigator.pushNamed`** — bypasses go_router's route tree,
   breaking deep linking, the browser back button on web, and auth redirect guards.
   Always use `context.go()`, `context.push()`, or `context.goNamed()`.

2. **Hardcoded path strings at call sites** — `context.go('/products/123')` scattered
   across widgets is fragile. Use named routes with `pathParameters`.

3. **Auth logic inside screen widgets** — checking auth state inside `build` and
   redirecting manually creates race conditions. Use the `redirect` callback.

4. **`BuildContext` passed across async gaps without `mounted` check** — always guard
   context usage after `await`:
   ```dart
   await someAsyncOperation();
   if (!context.mounted) return;
   context.go('/success');
   ```
