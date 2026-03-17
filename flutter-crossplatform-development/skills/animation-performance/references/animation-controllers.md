---
description: AnimationController, Tweens, implicit/explicit animation patterns, staggered animations, Hero, Rive/Lottie
---

# Animation Controllers Reference

## 1. Refresh Rate Awareness

Flutter automatically synchronizes with the device's display refresh rate (e.g., 120 Hz on
ProMotion displays). The engine drives the `Ticker`; your code only declares duration and curve.

**Rule**: Never artificially cap or drive FPS manually. Let the engine control the ticker.

```dart
// ✅ CORRECT — Flutter manages vsync automatically
AnimationController(
  vsync: this,
  duration: const Duration(milliseconds: 300),
);

// ❌ WRONG — Manual timer is not vsync-synchronized, wastes CPU, causes tearing
Timer.periodic(const Duration(milliseconds: 16), (timer) {
  setState(() { _value += 0.016; });
});
```

## 2. Avoid Widget Tree Rebuilds

**Problem**: Calling `setState` inside an animation listener marks the entire subtree as dirty
and re-runs the build phase — at up to 120 Hz. This is almost always unnecessary.

**Rule**: Use `AnimatedBuilder` (or `AnimatedWidget`) to scope rebuilds to the smallest
possible subtree. Pass expensive, stable children as the `child` argument so they are built
once and reused.

```dart
// ❌ WRONG — setState rebuilds everything above the listener
_controller.addListener(() {
  setState(() { _value = _controller.value; });
});

// ✅ CORRECT — AnimatedBuilder scopes the dirty region
AnimatedBuilder(
  animation: _controller,
  builder: (context, child) => Transform.translate(
    offset: Offset(_controller.value * 100, 0),
    child: child,           // child is built once, not on every frame
  ),
  child: const ExpensiveWidget(),
)
```

## 10. Implicit vs. Explicit Animation Patterns

### Implicit Animations — Use for Simple Property Changes

Prefer Flutter's built-in implicit widgets when you only need to animate a single property
in response to a state change. No controller, no `dispose()` required.

```dart
// Animate size
AnimatedContainer(
  duration: const Duration(milliseconds: 300),
  curve: Curves.easeInOut,
  width: _expanded ? 200.0 : 100.0,
  height: _expanded ? 200.0 : 100.0,
  child: const FlutterLogo(),
)

// Animate opacity
AnimatedOpacity(
  duration: const Duration(milliseconds: 300),
  opacity: _visible ? 1.0 : 0.0,
  child: child,
)

// Other implicit widgets: AnimatedPadding, AnimatedAlign, AnimatedDefaultTextStyle,
// AnimatedPhysicalModel, AnimatedPositioned (inside Stack), TweenAnimationBuilder
```

### Explicit Animations — Use for Precise Control

When you need to sequence, reverse, repeat, or compose multiple animations on different
properties, use `AnimationController` + `CurvedAnimation` + `Tween`.

**Always dispose the controller.**

```dart
class ComplexAnimation extends StatefulWidget {
  const ComplexAnimation({super.key});

  @override
  State<ComplexAnimation> createState() => _ComplexAnimationState();
}

class _ComplexAnimationState extends State<ComplexAnimation>
    with SingleTickerProviderStateMixin {
  late final AnimationController _controller;
  late final Animation<double> _fade;
  late final Animation<Offset> _slide;

  @override
  void initState() {
    super.initState();
    _controller = AnimationController(
      vsync: this,
      duration: const Duration(milliseconds: 600),
    );

    _fade = Tween<double>(begin: 0.0, end: 1.0).animate(
      CurvedAnimation(parent: _controller, curve: Curves.easeIn),
    );

    _slide = Tween<Offset>(
      begin: const Offset(0.0, 1.0),
      end: Offset.zero,
    ).animate(CurvedAnimation(parent: _controller, curve: Curves.easeOut));

    _controller.forward();
  }

  @override
  Widget build(BuildContext context) {
    return RepaintBoundary(
      child: SlideTransition(
        position: _slide,
        child: FadeTransition(
          opacity: _fade,
          child: const MyContent(),
        ),
      ),
    );
  }

  @override
  void dispose() {
    _controller.dispose(); // ✅ Always dispose
    super.dispose();
  }
}
```

### Staggered Animations — Multiple Tweens with Interval

Use `Interval` as the curve to sequence multiple tweens from a single controller.

```dart
class _StaggeredState extends State<StaggeredDemo>
    with SingleTickerProviderStateMixin {
  late final AnimationController _controller;
  late final Animation<double> _opacity;
  late final Animation<double> _scale;
  late final Animation<Offset> _position;

  @override
  void initState() {
    super.initState();
    _controller = AnimationController(
      vsync: this,
      duration: const Duration(milliseconds: 900),
    );

    // Each tween fires during its own time slice (0.0–1.0 of the total duration)
    _opacity = Tween<double>(begin: 0.0, end: 1.0).animate(
      CurvedAnimation(
        parent: _controller,
        curve: const Interval(0.0, 0.4, curve: Curves.easeIn),
      ),
    );

    _scale = Tween<double>(begin: 0.5, end: 1.0).animate(
      CurvedAnimation(
        parent: _controller,
        curve: const Interval(0.2, 0.7, curve: Curves.elasticOut),
      ),
    );

    _position = Tween<Offset>(
      begin: const Offset(0.0, 0.3),
      end: Offset.zero,
    ).animate(
      CurvedAnimation(
        parent: _controller,
        curve: const Interval(0.4, 1.0, curve: Curves.easeOut),
      ),
    );

    _controller.forward();
  }

  @override
  Widget build(BuildContext context) {
    return AnimatedBuilder(
      animation: _controller,
      builder: (context, child) {
        return FadeTransition(
          opacity: _opacity,
          child: ScaleTransition(
            scale: _scale,
            child: SlideTransition(
              position: _position,
              child: child,
            ),
          ),
        );
      },
      child: const MyCard(), // built once
    );
  }

  @override
  void dispose() {
    _controller.dispose();
    super.dispose();
  }
}
```

### Hero Animations — Shared Element Transitions

`Hero` handles shared-element transitions automatically between routes. Tag the widget with
the same `tag` value on both source and destination screens.

```dart
// Source screen
Hero(
  tag: 'product-image-${product.id}',
  child: Image.network(product.imageUrl, width: 80, height: 80, fit: BoxFit.cover),
)

// Destination screen (same tag)
Hero(
  tag: 'product-image-${product.id}',
  child: Image.network(product.imageUrl, width: double.infinity, fit: BoxFit.cover),
)
```

For custom flight animation, provide a `flightShuttleBuilder`. To prevent layout recalculation
during the flight, avoid wrapping `Hero` in widgets that change size based on the hero's size.

### Rive / Lottie Integration

Both packages drive their own internal `AnimationController`-compatible ticker. Let them
manage playback; only interact with named state machines or input properties.

```dart
// Rive — StateMachine-based
RiveAnimation.asset(
  'assets/animations/button.riv',
  stateMachines: const ['ButtonStateMachine'],
  onInit: (artboard) {
    final controller = StateMachineController.fromArtboard(
      artboard,
      'ButtonStateMachine',
    );
    if (controller != null) {
      artboard.addController(controller);
      _isHovered = controller.findInput<bool>('isHovered') as SMIBool?;
    }
  },
)

// Lottie — plays JSON animation, optionally drive with your own controller
Lottie.asset(
  'assets/animations/loading.json',
  controller: _controller, // optional: sync with your AnimationController
  onLoaded: (composition) {
    _controller
      ..duration = composition.duration
      ..repeat();
  },
)
```

**Rule**: Always `dispose` any `AnimationController` you create for Lottie. Rive's internally
created controllers are disposed by the widget automatically.

## Dispose Rule

Every `AnimationController` and `Ticker` you create **must** be disposed in the `dispose()`
lifecycle method of its owning `State`.

```dart
@override
void dispose() {
  _primaryController.dispose();
  _secondaryController.dispose();
  super.dispose();
}
```

Failing to dispose leaks the vsync binding, causes "disposed with active Ticker" exceptions,
and prevents garbage collection of the state object.
