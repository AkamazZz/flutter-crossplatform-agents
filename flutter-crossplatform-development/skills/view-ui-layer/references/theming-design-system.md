---
description: Theming and design system — Theme.of(context), Material 3 ColorScheme, TextTheme, dark/light mode, ThemeExtension, Cupertino theming, spacing/radius tokens.
---

# Theming & Design System

## 1. Theme.of(context) — Always (Global Rule 7)

**Never hardcode colours, text styles, or sizes.** Every visual token must come from
`Theme.of(context)` or its shorthand helpers. This ensures dark/light mode, dynamic
colour, and design-system changes propagate automatically.

```dart
// ❌ WRONG — hardcoded values
Text(
  'Hello',
  style: TextStyle(color: Color(0xFF1A73E8), fontSize: 16),
)

Container(
  color: Colors.white,
  child: Icon(Icons.star, color: Colors.amber),
)

// ✅ CORRECT — theme tokens
Text(
  'Hello',
  style: Theme.of(context).textTheme.bodyLarge?.copyWith(
    color: Theme.of(context).colorScheme.primary,
  ),
)

Container(
  color: Theme.of(context).colorScheme.surface,
  child: Icon(Icons.star, color: Theme.of(context).colorScheme.tertiary),
)
```

Shorthand helpers (Flutter 3.x+):

```dart
final colorScheme = Theme.of(context).colorScheme;
final textTheme   = Theme.of(context).textTheme;
```

---

## 2. Material 3 ColorScheme

Material 3 uses a role-based colour system generated from a seed colour. The project
defines its `ThemeData` with a `ColorScheme.fromSeed()`.

```dart
// lib/core/theme/app_theme.dart
import 'package:flutter/material.dart';

abstract final class AppTheme {
  static ThemeData get light => ThemeData(
    useMaterial3: true,
    colorScheme: ColorScheme.fromSeed(
      seedColor: const Color(0xFF1A73E8),
      brightness: Brightness.light,
    ),
    textTheme: _textTheme,
  );

  static ThemeData get dark => ThemeData(
    useMaterial3: true,
    colorScheme: ColorScheme.fromSeed(
      seedColor: const Color(0xFF1A73E8),
      brightness: Brightness.dark,
    ),
    textTheme: _textTheme,
  );
}
```

Key colour roles and their intended use:

| Role | Use |
|---|---|
| `primary` | Primary action buttons, active icons, key interactive elements |
| `onPrimary` | Content (text/icons) on top of `primary` |
| `secondary` | Supporting interactive elements |
| `surface` | Card, sheet, menu backgrounds |
| `onSurface` | Standard body text on surfaces |
| `error` | Error states, destructive actions |
| `onError` | Text/icons on `error` colour |
| `outline` | Borders, dividers |
| `inverseSurface` | Snackbar, tooltip backgrounds |

### Dynamic Theming (Android 12+)

```dart
import 'package:dynamic_color/dynamic_color.dart';

DynamicColorBuilder(
  builder: (lightDynamic, darkDynamic) {
    return MaterialApp.router(
      theme: AppTheme.lightFrom(lightDynamic),
      darkTheme: AppTheme.darkFrom(darkDynamic),
      routerConfig: appRouter,
    );
  },
)
```

```dart
// In app_theme.dart
static ThemeData lightFrom(ColorScheme? dynamic) => ThemeData(
  useMaterial3: true,
  colorScheme: dynamic ?? ColorScheme.fromSeed(
    seedColor: const Color(0xFF1A73E8),
    brightness: Brightness.light,
  ),
);
```

---

## 3. TextTheme Typography

Use the named text roles. Never create free-standing `TextStyle` objects in widgets.

| Role | Usage |
|---|---|
| `displayLarge/Medium/Small` | Hero numbers, splash screens |
| `headlineLarge/Medium/Small` | Screen titles, section headers |
| `titleLarge/Medium/Small` | List item titles, app bar |
| `bodyLarge/Medium/Small` | Body copy, descriptions |
| `labelLarge/Medium/Small` | Button labels, chips, captions |

```dart
// ✅ CORRECT
Text(product.name, style: theme.textTheme.titleMedium)
Text(product.description, style: theme.textTheme.bodyMedium)

// Use copyWith only to layer on state-specific overrides
Text(
  state.errorMessage,
  style: theme.textTheme.bodyMedium?.copyWith(
    color: theme.colorScheme.error,
  ),
)
```

Define a custom `TextTheme` in `app_theme.dart` if the project uses a non-default
type scale:

```dart
static const TextTheme _textTheme = TextTheme(
  headlineLarge: TextStyle(fontFamily: 'Inter', fontWeight: FontWeight.w700),
  bodyMedium:    TextStyle(fontFamily: 'Inter', fontWeight: FontWeight.w400),
);
```

---

## 4. Dark / Light Theme in MaterialApp

```dart
MaterialApp.router(
  theme:      AppTheme.light,
  darkTheme:  AppTheme.dark,
  themeMode:  ThemeMode.system,   // follows OS setting
  routerConfig: appRouter,
)
```

To allow in-app theme switching, store the selected `ThemeMode` in a BLoC or
`ValueNotifier` and pass it to `themeMode`:

```dart
BlocBuilder<ThemeBloc, ThemeState>(
  builder: (context, state) {
    return MaterialApp.router(
      theme:      AppTheme.light,
      darkTheme:  AppTheme.dark,
      themeMode:  state.themeMode,
      routerConfig: appRouter,
    );
  },
)
```

---

## 5. ThemeExtension for App-Specific Tokens

Material 3's built-in colour roles do not cover every app-specific design token (e.g.
success colour, skeleton shimmer colour, card border radius). Use `ThemeExtension` to add
custom tokens that participate in theme interpolation.

```dart
// lib/core/theme/app_colors_extension.dart
@immutable
class AppColorsExtension extends ThemeExtension<AppColorsExtension> {
  const AppColorsExtension({
    required this.success,
    required this.onSuccess,
    required this.warning,
    required this.shimmer,
  });

  final Color success;
  final Color onSuccess;
  final Color warning;
  final Color shimmer;

  @override
  AppColorsExtension copyWith({
    Color? success,
    Color? onSuccess,
    Color? warning,
    Color? shimmer,
  }) {
    return AppColorsExtension(
      success:   success   ?? this.success,
      onSuccess: onSuccess ?? this.onSuccess,
      warning:   warning   ?? this.warning,
      shimmer:   shimmer   ?? this.shimmer,
    );
  }

  @override
  AppColorsExtension lerp(AppColorsExtension? other, double t) {
    if (other == null) return this;
    return AppColorsExtension(
      success:   Color.lerp(success,   other.success,   t)!,
      onSuccess: Color.lerp(onSuccess, other.onSuccess, t)!,
      warning:   Color.lerp(warning,   other.warning,   t)!,
      shimmer:   Color.lerp(shimmer,   other.shimmer,   t)!,
    );
  }
}
```

Register extensions in `ThemeData`:

```dart
static ThemeData get light => ThemeData(
  useMaterial3: true,
  colorScheme: ...,
  extensions: const [
    AppColorsExtension(
      success:   Color(0xFF34A853),
      onSuccess: Colors.white,
      warning:   Color(0xFFFBBC05),
      shimmer:   Color(0xFFE0E0E0),
    ),
    AppSpacingExtension(...),
  ],
);
```

Access in widgets:

```dart
final appColors = Theme.of(context).extension<AppColorsExtension>()!;

Icon(Icons.check_circle, color: appColors.success)
```

---

## 6. Design System Tokens — Spacing, Radius, Elevation

Define spacing, border radius, and elevation as `ThemeExtension` values so they are
theme-aware and easily updated globally.

```dart
@immutable
class AppSpacingExtension extends ThemeExtension<AppSpacingExtension> {
  const AppSpacingExtension({
    required this.xs,
    required this.sm,
    required this.md,
    required this.lg,
    required this.xl,
  });

  final double xs;   //  4
  final double sm;   //  8
  final double md;   // 16
  final double lg;   // 24
  final double xl;   // 32

  @override
  AppSpacingExtension copyWith({...}) => ...;

  @override
  AppSpacingExtension lerp(AppSpacingExtension? other, double t) => ...;
}

@immutable
class AppShapeExtension extends ThemeExtension<AppShapeExtension> {
  const AppShapeExtension({
    required this.small,
    required this.medium,
    required this.large,
    required this.full,
  });

  final BorderRadius small;   // 4
  final BorderRadius medium;  // 12
  final BorderRadius large;   // 24
  final BorderRadius full;    // circular

  @override
  AppShapeExtension copyWith({...}) => ...;

  @override
  AppShapeExtension lerp(AppShapeExtension? other, double t) => ...;
}
```

Usage:

```dart
final spacing = Theme.of(context).extension<AppSpacingExtension>()!;
final shape   = Theme.of(context).extension<AppShapeExtension>()!;

Padding(
  padding: EdgeInsets.all(spacing.md),
  child: Card(
    shape: RoundedRectangleBorder(borderRadius: shape.medium),
    child: ...,
  ),
)
```

---

## 7. Cupertino Theming

When rendering Cupertino widgets, provide a `CupertinoThemeData` inside
`CupertinoApp` or wrap the Cupertino subtree with `CupertinoTheme`.

```dart
// Standalone Cupertino app
CupertinoApp(
  theme: const CupertinoThemeData(
    primaryColor: CupertinoColors.activeBlue,
    brightness: Brightness.light,
    textTheme: CupertinoTextThemeData(
      navTitleTextStyle: TextStyle(
        fontFamily: 'SF Pro',
        fontSize: 17,
        fontWeight: FontWeight.w600,
      ),
    ),
  ),
  home: const HomeScreen(),
)
```

Mixing Material and Cupertino in one app:

```dart
// Wrap the Cupertino subtree
CupertinoTheme(
  data: CupertinoThemeData(
    primaryColor: Theme.of(context).colorScheme.primary,
  ),
  child: CupertinoSlidingSegmentedControl(
    groupValue: selectedSegment,
    children: const {0: Text('One'), 1: Text('Two')},
    onValueChanged: (v) => ...,
  ),
)
```
