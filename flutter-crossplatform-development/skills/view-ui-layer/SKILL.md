---
name: view-ui-layer
description: >
  Flutter View/UI layer patterns with BlocBuilder, BlocListener, widget composition, and
  responsive design. Use when writing screens, widgets, connecting UI to BLoC, theming, or
  building responsive layouts. Trigger on: screens, widgets, BlocBuilder, BlocListener,
  BlocConsumer, navigation, theming, responsive, adaptive, Material 3, Cupertino, go_router.
---

# Flutter View / UI Layer Skill

This skill enforces the project's presentation-layer standards. Before answering, read the
relevant reference file(s) for the specific topic.

## Architecture Overview

The View layer is the outermost shell of the feature slice. Its sole responsibilities are:

- Rendering widgets in response to BLoC state emissions.
- Forwarding user gestures as events into the BLoC (`add()`).
- Triggering side effects (navigation, snackbars, dialogs) via `BlocListener`.
- Adapting layout to screen size and platform.
- Consuming theme tokens — never hardcoding colours, sizes, or strings.

```
User Gesture
    │
    ▼
[View / Screen]  ──add(event)──▶  [BLoC]
    │                                │
    │◀──────── state emission ────────┘
    │
    ├── BlocBuilder  → rebuild widgets
    └── BlocListener → side effects
```

The View layer must **not** contain business logic, repository calls, or direct data-source
access. All such concerns belong to the BLoC or Domain layers.

Directory structure: see CLAUDE.md — Directory Structure Convention.

## Reference Files — Read Before Answering

| Topic | Reference file | When to read |
|---|---|---|
| Widget composition | `references/widget-composition.md` | Extracting build methods, const constructors, Key strategy, private widget naming |
| BLoC widgets | `references/bloc-widgets.md` | BlocBuilder, BlocListener, BlocConsumer, context extensions, buildWhen/listenWhen |
| Responsive & adaptive | `references/responsive-adaptive.md` | LayoutBuilder, MediaQuery, breakpoints, platform-adaptive widgets |
| Navigation | `references/navigation.md` | go_router setup, route tree, deep linking, auth guards |
| Theming & design system | `references/theming-design-system.md` | Theme.of(context), Material 3, dark/light mode, ThemeExtension |

**Rule**: Always read the relevant reference(s) in full before writing View-layer code.
Multiple references often apply — e.g. a new screen needs `bloc-widgets.md`,
`widget-composition.md`, and `theming-design-system.md`.

## Quick Reference — View Layer Rules

| Concern | Rule |
|---|---|
| Widget size | Small and focused; extract private widget classes (`_FeatureHeader`, `_FeatureListItem`) |
| Statefulness | `StatelessWidget` by default; `StatefulWidget` only for ephemeral state (animations, text controllers) |
| State rendering | `BlocBuilder` + `switch` expression on sealed states — exhaustive, no fallback `_` |
| Side effects | `BlocListener` only — never trigger navigation or snackbars inside `builder` |
| Both rendering + side effects | `BlocConsumer` |
| Boilerplate | Context extensions for repeated `context.read<Bloc>().add(...)` calls |
| Rebuild optimisation | `buildWhen` / `listenWhen` to gate unnecessary rebuilds |
| Styling | `Theme.of(context)` always — never hardcode colours, text styles, or sizes |
| Strings | Localisation classes only — never inline string literals visible to users |

## Key Anti-Patterns

1. **Stateful by default** — using `StatefulWidget` when no ephemeral state is needed
   adds unnecessary overhead and complexity.

2. **Monolithic `build` methods** — a `build` method exceeding ~40 lines is a signal to
   extract private widget classes.

3. **Business logic in widgets** — filtering lists, computing derived values, or calling
   APIs directly inside `build` or gesture callbacks.

4. **Hardcoded colours / text styles** — `Color(0xFF...)` or `TextStyle(color: Colors.red)`
   instead of `Theme.of(context).colorScheme.*` or `Theme.of(context).textTheme.*`.

5. **Inline string literals** — user-visible strings typed directly in widget code instead
   of localisation class accessors.

6. **`Navigator.push` / `Navigator.pushNamed`** — bypasses go_router's route tree,
   breaking deep linking and auth guards.

7. **Side effects in `builder`** — calling `Navigator`, `ScaffoldMessenger`, or other
   imperative APIs inside `BlocBuilder.builder`; use `BlocListener` instead.

8. **Missing `const` constructors** — forgetting `const` on widgets with no runtime
   parameters causes avoidable rebuilds.

9. **Wildcard `_` in switch on sealed states** — hides unhandled new states at compile
   time; always enumerate all cases explicitly.

10. **`context.watch<Bloc>()` inside deep children** — subscribes the whole subtree;
    prefer `BlocBuilder` scoped to the smallest widget that needs the state.
