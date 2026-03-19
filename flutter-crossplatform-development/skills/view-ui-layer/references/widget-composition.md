---
description: Widget composition rules — small focused widgets, statefulness, const constructors, Key strategy, private naming.
---

# Widget Composition

## 1. Small, Focused Widgets

A `build` method should do one thing. When a method grows beyond ~40 lines or nests
more than 3–4 levels deep, extract sub-components into private widget classes.

### Before — monolithic build

```dart
class ProductScreen extends StatelessWidget {
  const ProductScreen({super.key, required this.product});

  final Product product;

  @override
  Widget build(BuildContext context) {
    final theme = Theme.of(context);
    return Scaffold(
      body: Column(
        crossAxisAlignment: CrossAxisAlignment.start,
        children: [
          Image.network(product.imageUrl, height: 240, fit: BoxFit.cover),
          Padding(
            padding: const EdgeInsets.all(16),
            child: Text(product.name, style: theme.textTheme.headlineSmall),
          ),
          Padding(
            padding: const EdgeInsets.symmetric(horizontal: 16),
            child: Text(product.description, style: theme.textTheme.bodyMedium),
          ),
          const Divider(),
          Padding(
            padding: const EdgeInsets.all(16),
            child: Row(
              children: [
                Text('\$${product.price}', style: theme.textTheme.titleLarge),
                const Spacer(),
                FilledButton(
                  onPressed: () => context.read<CartBloc>().add(
                    CartEvent.addItem(product),
                  ),
                  child: const Text('Add to Cart'),
                ),
              ],
            ),
          ),
        ],
      ),
    );
  }
}
```

### After — extracted private widget classes

```dart
class ProductScreen extends StatelessWidget {
  const ProductScreen({super.key, required this.product});

  final Product product;

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: Column(
        crossAxisAlignment: CrossAxisAlignment.start,
        children: [
          _ProductImage(imageUrl: product.imageUrl),
          _ProductDetails(product: product),
          const Divider(),
          _ProductActions(product: product),
        ],
      ),
    );
  }
}

class _ProductImage extends StatelessWidget {
  const _ProductImage({required this.imageUrl});

  final String imageUrl;

  @override
  Widget build(BuildContext context) {
    return Image.network(imageUrl, height: 240, fit: BoxFit.cover);
  }
}

class _ProductDetails extends StatelessWidget {
  const _ProductDetails({required this.product});

  final Product product;

  @override
  Widget build(BuildContext context) {
    final theme = Theme.of(context);
    return Padding(
      padding: const EdgeInsets.all(16),
      child: Column(
        crossAxisAlignment: CrossAxisAlignment.start,
        children: [
          Text(product.name, style: theme.textTheme.headlineSmall),
          const SizedBox(height: 8),
          Text(product.description, style: theme.textTheme.bodyMedium),
        ],
      ),
    );
  }
}

class _ProductActions extends StatelessWidget {
  const _ProductActions({required this.product});

  final Product product;

  @override
  Widget build(BuildContext context) {
    final theme = Theme.of(context);
    return Padding(
      padding: const EdgeInsets.all(16),
      child: Row(
        children: [
          Text('\$${product.price}', style: theme.textTheme.titleLarge),
          const Spacer(),
          FilledButton(
            onPressed: () => context.read<CartBloc>().add(
              CartEvent.addItem(product),
            ),
            child: const Text('Add to Cart'),
          ),
        ],
      ),
    );
  }
}
```

Benefits:
- Each class has a single responsibility and is individually testable.
- Flutter's element reconciliation can skip unchanged subtrees more aggressively.
- `const` constructors become possible once all fields are final.

---

## 2. StatelessWidget by Default

**Rule (Global Rule 9)**: Default to `StatelessWidget`. Promote to `StatefulWidget` only
when the widget itself owns ephemeral (transient, non-shared) state.

- BLoC-driven state (loading, data, error) → `StatelessWidget` + `BlocBuilder`
- Animation controller → `StatefulWidget` (or `AnimationWidget`)
- `TextEditingController` → `StatefulWidget`
- `FocusNode` → `StatefulWidget`
- Form validation feedback → `StatefulWidget`
- Scroll offset tracking → `StatefulWidget` or `ScrollController` in parent

```dart
// ✅ CORRECT — ephemeral text controller warrants StatefulWidget
class SearchField extends StatefulWidget {
  const SearchField({super.key, required this.onSubmit});

  final ValueChanged<String> onSubmit;

  @override
  State<SearchField> createState() => _SearchFieldState();
}

class _SearchFieldState extends State<SearchField> {
  late final TextEditingController _controller;

  @override
  void initState() {
    super.initState();
    _controller = TextEditingController();
  }

  @override
  void dispose() {
    _controller.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return TextField(
      controller: _controller,
      onSubmitted: widget.onSubmit,
    );
  }
}

// ✅ CORRECT — no ephemeral state needed
class ProductTitle extends StatelessWidget {
  const ProductTitle({super.key, required this.title});

  final String title;

  @override
  Widget build(BuildContext context) {
    return Text(title, style: Theme.of(context).textTheme.headlineSmall);
  }
}
```

---

## 3. Const Constructors

Mark constructors `const` whenever all fields are `final` and no non-const default values
are used. Flutter's framework skips rebuilding `const` widgets whose instance identity is
stable.

```dart
// ✅ CORRECT
class _LoadingIndicator extends StatelessWidget {
  const _LoadingIndicator();   // no key needed for private widgets

  @override
  Widget build(BuildContext context) {
    return const Center(child: CircularProgressIndicator());
  }
}

// Usage — the const keyword at call site is the performance win
return const _LoadingIndicator();
```

Rules:
- Every `StatelessWidget` with no constructor parameters should be `const`.
- Use `const` at instantiation site wherever possible (`const SizedBox(height: 16)`).
- Linter rule `prefer_const_constructors` should be enabled in `analysis_options.yaml`.

---

## 4. Key Strategy

Keys help Flutter reconcile the widget tree when order or type changes. Use the correct
key type for each scenario.

- `ValueKey<T>` — list items with a stable domain identifier (e.g. `ValueKey(product.id)`)
- `ObjectKey` — list items where the whole object is the identity (e.g. `ObjectKey(address)`)
- `UniqueKey` — force-recreate a widget on every build; use sparingly, breaks animations
- `GlobalKey` — access `State` or `RenderObject` from outside the tree; avoid in most cases
- No key — default for static, non-reordering widget trees

```dart
// ✅ CORRECT — stable domain key for animated list
ListView.builder(
  itemCount: products.length,
  itemBuilder: (context, index) {
    final product = products[index];
    return _ProductListItem(
      key: ValueKey(product.id),
      product: product,
    );
  },
)

// ✅ CORRECT — force widget recreation when switching between incompatible configs
AnimatedSwitcher(
  duration: const Duration(milliseconds: 300),
  child: isLoggedIn
      ? const _LoggedInView(key: ValueKey('logged-in'))
      : const _GuestView(key: ValueKey('guest')),
)

// ❌ WRONG — UniqueKey in build method recreates widget every frame
return SomeWidget(key: UniqueKey()); // causes constant disposal/recreation
```

---

## 5. Private Widget Class Naming

File-private widget classes (declared with `_`) follow this convention:

```
_[FeatureName][Role]
```

- `_[Feature]Header` — e.g. `_ProductHeader`
- `_[Feature]ListItem` — e.g. `_OrderListItem`
- `_[Feature]Actions` — e.g. `_ProfileActions`
- `_[Feature]EmptyState` — e.g. `_SearchEmptyState`
- `_[Feature]ErrorView` — e.g. `_CheckoutErrorView`
- `_[State]View` — e.g. `_LoadingView`, `_SuccessView`, `_ErrorView`

Rules:
- Prefix with `_` when the widget is used only within the same file.
- Move to `widgets/` directory and drop the `_` prefix when shared across screens.
- Do not abbreviate feature names — clarity beats brevity.
