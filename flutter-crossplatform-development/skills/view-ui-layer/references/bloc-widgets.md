---
description: BLoC widget patterns — BlocBuilder, BlocListener, BlocConsumer, context extensions, buildWhen/listenWhen, MultiBlocProvider/Listener.
---

# BLoC Widgets

> Cross-reference: For BLoC class authoring (events, states, transformers) see the
> `bloc-state-management` skill.

## 1. BlocBuilder with Switch Expression on Sealed States

`BlocBuilder` rebuilds its subtree whenever the BLoC emits a new state. Use a `switch`
expression — never `if/else` chains — to guarantee exhaustive handling of every sealed
state variant. The compiler will error if a new state is added but not handled.

```dart
class FeatureScreen extends StatelessWidget {
  const FeatureScreen({super.key});

  @override
  Widget build(BuildContext context) {
    return BlocBuilder<FeatureBloc, FeatureState>(
      builder: (context, state) {
        // Shared base properties are accessible without casting
        final currentItems = state.items;
        final hasMore = state.hasMore;

        return switch (state) {
          FeatureStateIdle() =>
            Column(
              children: [
                Text('Items: ${currentItems.length}'),
                if (hasMore)
                  FilledButton(
                    onPressed: () => context.read<FeatureBloc>().add(
                      const FeatureEvent.loadMore(),
                    ),
                    child: const Text('Load More'),
                  ),
              ],
            ),

          FeatureStateProcessing() =>
            const Center(child: CircularProgressIndicator()),

          FeatureStateSuccessful() =>
            ListView.builder(
              itemCount: state.items.length,
              itemBuilder: (context, index) =>
                _FeatureListItem(item: state.items[index]),
            ),

          // Destructuring extracts the field directly
          FeatureStateError(:final message) =>
            _ErrorView(message: message, onRetry: () {
              context.read<FeatureBloc>().add(const FeatureEvent.retry());
            }),
        };
      },
    );
  }
}
```

Key rules:
- Never use `_` wildcard — always enumerate all cases explicitly.
- Prefer delegating each case to a private widget class (`_SuccessView`, `_ErrorView`)
  rather than inlining long widget trees inside the switch arms.
- Access shared base-class properties (defined on the sealed base) without casting;
  use destructuring (`:final field`) for subtype-specific properties.

---

## 2. BlocListener for Side Effects

`BlocListener` responds to state changes without rebuilding the widget. Use it for
navigation, snackbars, dialogs, haptic feedback, or any other imperative side effect.

**Rule**: Never trigger navigation or `ScaffoldMessenger` calls inside `BlocBuilder.builder`.

```dart
class FeatureScreen extends StatelessWidget {
  const FeatureScreen({super.key});

  @override
  Widget build(BuildContext context) {
    return BlocListener<FeatureBloc, FeatureState>(
      listener: (context, state) => switch (state.action) {
        final ShowErrorAction action   => _showError(context, action),
        final NavigateAwayAction action => _navigateAway(context, action),
        null => null,  // no action pending
      },
      child: BlocBuilder<FeatureBloc, FeatureState>(
        builder: (context, state) => switch (state) {
          FeatureStateIdle()       => const _IdleView(),
          FeatureStateProcessing() => const _ProcessingView(),
          FeatureStateSuccessful() => _SuccessView(state: state),
          FeatureStateError()      => const _ErrorView(),
        },
      ),
    );
  }

  void _showError(BuildContext context, ShowErrorAction action) {
    ScaffoldMessenger.of(context).showSnackBar(
      SnackBar(content: Text(action.message)),
    );
  }

  void _navigateAway(BuildContext context, NavigateAwayAction action) {
    context.go(action.route);
  }
}
```

Pattern matching rules for `listener`:
- Delegate to focused private methods (`_showError`, `_navigateAway`).
- Handle the `null` case explicitly to suppress linter warnings.
- Keep side-effect methods short — a method that does more than one thing should be split.

---

## 3. BlocConsumer — Builder + Listener Combined

Use `BlocConsumer` when the same state transition must both rebuild the UI **and** trigger
a side effect. Avoid using it when only one behaviour is needed.

```dart
BlocConsumer<FeatureBloc, FeatureState>(
  listenWhen: (previous, current) =>
      current.action != null && current.action != previous.action,
  listener: (context, state) => switch (state.action) {
    final ShowErrorAction action    => _showError(context, action),
    final NavigateAwayAction action => _navigateAway(context, action),
    null => null,
  },
  buildWhen: (previous, current) => previous.runtimeType != current.runtimeType,
  builder: (context, state) => switch (state) {
    FeatureStateIdle()       => const _IdleView(),
    FeatureStateProcessing() => const _ProcessingView(),
    FeatureStateSuccessful() => _SuccessView(state: state),
    FeatureStateError()      => const _ErrorView(),
  },
)
```

---

## 4. Context Extensions to Reduce Boilerplate

Repeated `context.read<FeatureBloc>().add(...)` calls clutter widgets. Encapsulate them in
extension methods on `BuildContext`.

```dart
extension FeatureBlocX on BuildContext {
  FeatureBloc get featureBloc => read<FeatureBloc>();

  void loadFeatureData() =>
      read<FeatureBloc>().add(const FeatureEvent.load());

  void refreshFeatureData() =>
      read<FeatureBloc>().add(const FeatureEvent.refresh());

  void loadMoreFeatureData() =>
      read<FeatureBloc>().add(const FeatureEvent.loadMore());
}
```

Usage in a widget:

```dart
FilledButton(
  onPressed: context.loadFeatureData,
  child: const Text('Load'),
)

RefreshIndicator(
  onRefresh: () async => context.refreshFeatureData(),
  child: _FeatureList(),
)
```

Rules:
- Place extensions in `lib/features/[feature]/view/[feature]_bloc_extensions.dart`.
- Only wrap `add()` calls — do not put logic or conditionals inside extensions.
- Do not expose `watch<>()` via extensions; use `BlocBuilder` for that.

---

## 5. buildWhen / listenWhen Optimisation

Gate rebuilds and listener invocations to avoid unnecessary work.

### buildWhen

```dart
BlocBuilder<FeatureBloc, FeatureState>(
  // Only rebuild when the items list reference changes; ignore processing-state noise
  buildWhen: (previous, current) => previous.items != current.items,
  builder: (context, state) {
    return ListView.builder(
      itemCount: state.items.length,
      itemBuilder: (context, index) =>
          _FeatureListItem(item: state.items[index]),
    );
  },
)
```

### listenWhen

```dart
BlocListener<FeatureBloc, FeatureState>(
  // Fire only on the first transition into error state
  listenWhen: (previous, current) =>
      current is FeatureStateError && previous is! FeatureStateError,
  listener: (context, state) {
    if (state is FeatureStateError) {
      ScaffoldMessenger.of(context).showSnackBar(
        SnackBar(content: Text(state.message)),
      );
    }
  },
  child: const _FeatureBody(),
)
```

Guidelines:
- Prefer equality comparisons (`previous.items != current.items`) over type checks where
  possible.
- For action-based listeners, compare action identity to prevent re-firing:
  `current.action != null && current.action != previous.action`.
- Do not perform expensive computation inside `buildWhen` / `listenWhen`.

---

## 6. MultiBlocProvider / MultiBlocListener

When a screen requires multiple BLoCs, use `MultiBlocProvider` and `MultiBlocListener`
instead of nesting individual providers.

```dart
// Providing multiple BLoCs
MultiBlocProvider(
  providers: [
    BlocProvider<AuthBloc>(
      create: (context) => AuthBloc(repository: context.read()),
    ),
    BlocProvider<FeatureBloc>(
      create: (context) => FeatureBloc(repository: context.read()),
    ),
  ],
  child: const FeatureScreen(),
)
```

```dart
// Listening to multiple BLoCs
MultiBlocListener(
  listeners: [
    BlocListener<AuthBloc, AuthState>(
      listener: (context, state) {
        if (state is AuthStateUnauthenticated) {
          context.go('/login');
        }
      },
    ),
    BlocListener<FeatureBloc, FeatureState>(
      listenWhen: (previous, current) =>
          current is FeatureStateError && previous is! FeatureStateError,
      listener: (context, state) {
        if (state is FeatureStateError) {
          ScaffoldMessenger.of(context).showSnackBar(
            SnackBar(content: Text(state.message)),
          );
        }
      },
    ),
  ],
  child: const _FeatureBody(),
)
```

---

## 7. Best Practices for BLoC in Views

1. **Access shared base properties without casting**
   ```dart
   // ✅ CORRECT — shared property on sealed base class
   Text('Total: ${state.items.length}')

   // ✅ CORRECT — destructure subtype property
   FeatureStateError(:final message) => _ErrorView(message: message)
   ```

2. **Separate state views into private widget classes**
   ```dart
   builder: (context, state) => switch (state) {
     FeatureStateIdle()       => const _IdleView(),
     FeatureStateProcessing() => const _ProcessingView(),
     FeatureStateSuccessful() => _SuccessView(state: state),
     FeatureStateError()      => _ErrorView(state: state),
   },
   ```

3. **Use state helper getters for multi-place checks**
   ```dart
   // In the sealed state base class:
   bool get isProcessing => this is FeatureStateProcessing;

   // In the widget (when a single bool drives a small conditional):
   if (state.isProcessing) return const _ProcessingOverlay();
   ```

4. **Exhaust all sealed cases**
   Never use `_ =>` as a catch-all in `BlocBuilder` switch expressions. If a new state
   is added, the compiler must force you to handle it.

5. **Never call `context.read<Bloc>()` inside `builder`**
   Reading a BLoC inside `builder` does not subscribe to it. Use
   `context.read<Bloc>().add(event)` only in gesture callbacks, not in the build path.
