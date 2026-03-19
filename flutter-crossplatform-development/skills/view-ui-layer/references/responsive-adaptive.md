---
description: Responsive and adaptive layout patterns — LayoutBuilder, MediaQuery, breakpoints, platform-adaptive widgets, OrientationBuilder, SafeArea.
---

# Responsive & Adaptive Design

## 1. LayoutBuilder — Parent-Constraint-Based Layout

Use `LayoutBuilder` when the layout decision depends on the space the **parent** offers,
not the overall screen size. This is the preferred tool for reusable widgets that must
adapt to their container.

```dart
class ResponsiveGrid extends StatelessWidget {
  const ResponsiveGrid({super.key, required this.items});

  final List<Widget> items;

  @override
  Widget build(BuildContext context) {
    return LayoutBuilder(
      builder: (context, constraints) {
        final columns = switch (constraints.maxWidth) {
          < 400  => 1,
          < 700  => 2,
          < 1000 => 3,
          _      => 4,
        };

        return GridView.builder(
          gridDelegate: SliverGridDelegateWithFixedCrossAxisCount(
            crossAxisCount: columns,
            crossAxisSpacing: 12,
            mainAxisSpacing: 12,
          ),
          itemCount: items.length,
          itemBuilder: (context, index) => items[index],
        );
      },
    );
  }
}
```

When to use `LayoutBuilder` vs `MediaQuery`:
- `LayoutBuilder` — widget adapts to its containing box (sidebars, cards, grids).
- `MediaQuery` — top-level screen decisions (bottom nav vs rail, full-bleed images).

---

## 2. MediaQuery — Screen-Level Breakpoints

Use `MediaQuery.sizeOf(context)` (not `.of(context).size`) to read screen dimensions.
The `sizeOf` variant only rebuilds when the size changes, not on every
`MediaQueryData` change (safe area, text scale, etc.).

```dart
class AppShell extends StatelessWidget {
  const AppShell({super.key, required this.body});

  final Widget body;

  @override
  Widget build(BuildContext context) {
    final width = MediaQuery.sizeOf(context).width;

    return switch (width) {
      < 600  => _MobileShell(body: body),
      < 1200 => _TabletShell(body: body),
      _      => _DesktopShell(body: body),
    };
  }
}
```

---

## 3. Breakpoint System

The project uses three named breakpoints:

- Mobile — `< 600 dp` — single column, bottom navigation bar
- Tablet — `600 – 1200 dp` — two columns, navigation rail
- Desktop — `> 1200 dp` — multi-column, permanent navigation drawer

Centralise breakpoints in a constants file to avoid magic numbers:

```dart
// lib/core/layout/breakpoints.dart
abstract final class Breakpoints {
  static const double mobile  = 600;
  static const double tablet  = 1200;

  static bool isMobile(BuildContext context) =>
      MediaQuery.sizeOf(context).width < mobile;

  static bool isTablet(BuildContext context) {
    final w = MediaQuery.sizeOf(context).width;
    return w >= mobile && w < tablet;
  }

  static bool isDesktop(BuildContext context) =>
      MediaQuery.sizeOf(context).width >= tablet;
}
```

Usage:

```dart
NavigationBar vs NavigationRail example:

Widget build(BuildContext context) {
  if (Breakpoints.isMobile(context)) {
    return Scaffold(
      body: body,
      bottomNavigationBar: _AppBottomNavBar(),
    );
  }
  return Scaffold(
    body: Row(
      children: [
        _AppNavRail(),
        Expanded(child: body),
      ],
    ),
  );
}
```

---

## 4. Adaptive Widgets — Platform-Aware

Flutter's `adaptive` constructors and manual platform checks let you render
platform-native widgets on each OS.

```dart
import 'dart:io' show Platform;

class AdaptiveProgressIndicator extends StatelessWidget {
  const AdaptiveProgressIndicator({super.key});

  @override
  Widget build(BuildContext context) {
    return Platform.isIOS
        ? const CupertinoActivityIndicator()
        : const CircularProgressIndicator.adaptive();
  }
}

// Adaptive dialog
Future<bool?> showAdaptiveConfirm(BuildContext context, String message) {
  if (Platform.isIOS || Platform.isMacOS) {
    return showCupertinoDialog<bool>(
      context: context,
      builder: (_) => CupertinoAlertDialog(
        content: Text(message),
        actions: [
          CupertinoDialogAction(
            isDestructiveAction: true,
            onPressed: () => Navigator.pop(context, true),
            child: const Text('Confirm'),
          ),
          CupertinoDialogAction(
            onPressed: () => Navigator.pop(context, false),
            child: const Text('Cancel'),
          ),
        ],
      ),
    );
  }

  return showDialog<bool>(
    context: context,
    builder: (_) => AlertDialog(
      content: Text(message),
      actions: [
        TextButton(
          onPressed: () => Navigator.pop(context, false),
          child: const Text('Cancel'),
        ),
        FilledButton(
          onPressed: () => Navigator.pop(context, true),
          child: const Text('Confirm'),
        ),
      ],
    ),
  );
}
```

---

## 5. OrientationBuilder

Use `OrientationBuilder` to respond to device rotation without needing a full breakpoint
system.

```dart
class PhotoViewer extends StatelessWidget {
  const PhotoViewer({super.key, required this.photos});

  final List<String> photos;

  @override
  Widget build(BuildContext context) {
    return OrientationBuilder(
      builder: (context, orientation) {
        return GridView.builder(
          gridDelegate: SliverGridDelegateWithFixedCrossAxisCount(
            crossAxisCount: orientation == Orientation.portrait ? 2 : 4,
          ),
          itemCount: photos.length,
          itemBuilder: (context, index) =>
              Image.network(photos[index], fit: BoxFit.cover),
        );
      },
    );
  }
}
```

---

## 6. SafeArea

Wrap top-level screen content in `SafeArea` to avoid notch, status-bar, and home-indicator
overlap.

```dart
Scaffold(
  body: SafeArea(
    // bottom: false if Scaffold's bottomNavigationBar handles the inset
    bottom: false,
    child: _ScreenBody(),
  ),
  bottomNavigationBar: _AppBottomNavBar(),
)
```

Rules:
- Do not double-apply `SafeArea`; apply it once at the screen level.
- `Scaffold` does not apply `SafeArea` automatically — always add it explicitly.
- Disable `bottom: false` when a `Scaffold.bottomNavigationBar` is present; the bar
  handles the home-indicator inset itself.

---

## 7. Responsive Layout Patterns

### Single Column → Multi-Column

```dart
class ProductListScreen extends StatelessWidget {
  const ProductListScreen({super.key, required this.products});

  final List<Product> products;

  @override
  Widget build(BuildContext context) {
    return LayoutBuilder(
      builder: (context, constraints) {
        if (constraints.maxWidth < Breakpoints.mobile) {
          // Mobile: simple vertical list
          return ListView.builder(
            itemCount: products.length,
            itemBuilder: (context, index) =>
                _ProductListItem(product: products[index]),
          );
        }

        // Tablet+: master-detail two-column split
        return Row(
          children: [
            SizedBox(
              width: 320,
              child: ListView.builder(
                itemCount: products.length,
                itemBuilder: (context, index) =>
                    _ProductListItem(product: products[index]),
              ),
            ),
            const VerticalDivider(width: 1),
            const Expanded(child: _ProductDetailPanel()),
          ],
        );
      },
    );
  }
}
```

### Bottom Navigation → Side Navigation

```dart
class AdaptiveScaffold extends StatelessWidget {
  const AdaptiveScaffold({
    super.key,
    required this.selectedIndex,
    required this.onDestinationSelected,
    required this.body,
  });

  final int selectedIndex;
  final ValueChanged<int> onDestinationSelected;
  final Widget body;

  static const _destinations = [
    NavigationDestination(icon: Icon(Icons.home), label: 'Home'),
    NavigationDestination(icon: Icon(Icons.search), label: 'Search'),
    NavigationDestination(icon: Icon(Icons.person), label: 'Profile'),
  ];

  @override
  Widget build(BuildContext context) {
    final width = MediaQuery.sizeOf(context).width;

    if (width < Breakpoints.mobile) {
      return Scaffold(
        body: body,
        bottomNavigationBar: NavigationBar(
          selectedIndex: selectedIndex,
          onDestinationSelected: onDestinationSelected,
          destinations: _destinations,
        ),
      );
    }

    return Scaffold(
      body: Row(
        children: [
          NavigationRail(
            selectedIndex: selectedIndex,
            onDestinationSelected: onDestinationSelected,
            labelType: width >= Breakpoints.tablet
                ? NavigationRailLabelType.all
                : NavigationRailLabelType.selected,
            destinations: _destinations
                .map((d) => NavigationRailDestination(
                      icon: d.icon,
                      label: Text(d.label),
                    ))
                .toList(),
          ),
          const VerticalDivider(width: 1),
          Expanded(child: body),
        ],
      ),
    );
  }
}
```
