---
description: CustomPainter, RepaintBoundary isolation, canvas optimization, Paint reuse, batch drawing
---

# CustomPainter Reference

## 3. Isolate with RepaintBoundary

**Rule**: Wrap every animated `CustomPaint` (and any frequently-repainting widget) in a
`RepaintBoundary`.

**Why**: Flutter composites the UI as a tree of layers. A `RepaintBoundary` promotes the
subtree to its own compositing layer. When the animation triggers a repaint, only that
layer is re-rasterized — the rest of the screen (header, footer, navigation bar) stays
cached and is composited without redrawing.

```dart
// ✅ CORRECT — static siblings are not repainted
Column(
  children: [
    const StaticHeader(),            // cached, never repainted
    RepaintBoundary(
      child: AnimatedContent(),      // only this layer repaints
    ),
    const StaticFooter(),            // cached, never repainted
  ],
)
```

Do not add `RepaintBoundary` indiscriminately — each boundary allocates a separate GPU
texture. Use it where measured profiling shows excessive repainting of static siblings.

## 4. Maintain Structural Integrity During Animation

**Rule**: Never add, remove, or reorder widget tree nodes while an animation is running.

**Consequence**: Structural changes force a full layout pass on affected subtrees, which
competes with the ongoing paint cycle and causes jank (dropped frames).

To show or hide content during an animation, keep the node in the tree and animate its
visual properties instead.

```dart
// ❌ WRONG — conditionally inserting a node mid-animation
if (_controller.value > 0.5) const Text('Revealed'),

// ✅ CORRECT — node stays; only its opacity changes
Opacity(
  opacity: _controller.value > 0.5 ? 1.0 : 0.0,
  child: const Text('Revealed'),
)

// ✅ ALSO CORRECT — FadeTransition is more efficient (no widget rebuild)
FadeTransition(
  opacity: _controller,
  child: const Text('Revealed'),
)
```

## 5. The CustomPainter Pattern

**Rule**: Pass the `AnimationController` (or any `Listenable`) to the `repaint:` parameter
of `CustomPainter`. Never use `setState` to drive a `CustomPainter`.

**Why**: The `repaint` listenable triggers the paint phase directly, bypassing the build
phase entirely. Using `setState` instead rebuilds the entire widget tree above the
`CustomPaint` on every animation frame.

```dart
// ❌ WRONG — setState rebuilds the whole tree every frame
_controller.addListener(() {
  setState(() {}); // entire tree rebuilt at 60-120 Hz
});
// In build:
CustomPaint(painter: MyPainter(_controller.value))

// ✅ CORRECT — repaint: triggers paint phase only
RepaintBoundary(
  child: CustomPaint(
    painter: MyAnimatedPainter(animation: _controller),
  ),
)

class MyAnimatedPainter extends CustomPainter {
  MyAnimatedPainter({required this.animation}) : super(repaint: animation);

  final Animation<double> animation;

  @override
  void paint(Canvas canvas, Size size) {
    final progress = animation.value;
    // ... use progress to draw
  }

  @override
  bool shouldRepaint(MyAnimatedPainter oldDelegate) =>
      oldDelegate.animation != animation;
}
```

## 6. Paint Object Reuse

**Rule**: Define `Paint` objects once as class fields. Never instantiate `Paint` inside the
`paint()` method.

**Why**: `paint()` is called at every frame (60–120 Hz). Constructing a `Paint` object inside
the method allocates heap memory on every frame, adding GC pressure that can cause
micro-stutters.

```dart
// ❌ WRONG — allocates new Paint objects every frame
@override
void paint(Canvas canvas, Size size) {
  for (var i = 0; i < 100; i++) {
    final paint = Paint()..color = Colors.blue; // 100 allocations per frame!
    canvas.drawCircle(Offset(i * 10.0, i * 10.0), 5, paint);
  }
}

// ✅ CORRECT — single allocation at construction time
class EfficientPainter extends CustomPainter {
  final _fillPaint = Paint()
    ..color = Colors.blue
    ..style = PaintingStyle.fill;

  final _strokePaint = Paint()
    ..color = Colors.blueGrey
    ..style = PaintingStyle.stroke
    ..strokeWidth = 2.0;

  @override
  void paint(Canvas canvas, Size size) {
    for (var i = 0; i < 100; i++) {
      canvas.drawCircle(Offset(i * 10.0, i * 10.0), 5, _fillPaint);
    }
    canvas.drawRect(
      Rect.fromLTWH(0, 0, size.width, size.height),
      _strokePaint,
    );
  }

  @override
  bool shouldRepaint(EfficientPainter old) => false;
}
```

If you need to mutate color or other properties mid-paint, mutate the existing instance
(`_fillPaint.color = ...`) rather than creating a new one.

## 7. Batch Drawing Operations

Use `drawPoints` and `drawAtlas` to submit many shapes or sprites in a single GPU draw call.
Individual `drawCircle` / `drawRect` calls in a loop each generate a separate draw call,
increasing GPU command-buffer overhead.

```dart
// ❌ WRONG — N draw calls for N particles
for (final particle in particles) {
  canvas.drawCircle(particle.position, particle.radius, _paint);
}

// ✅ CORRECT — single draw call with drawPoints
final offsets = particles.map((p) => p.position).toList();
canvas.drawPoints(PointMode.points, offsets, _paint);

// ✅ BEST for sprites — drawAtlas submits all quads in one call
canvas.drawAtlas(
  spriteSheet,          // ui.Image atlas
  transforms,           // List<RSTransform> — position/scale/rotation per sprite
  rects,                // List<Rect> — source rect per sprite in the atlas
  null,                 // optional per-sprite colors
  BlendMode.src,
  null,                 // optional cull rect
  Paint(),
);
```

### Canvas Layer Management — save / restore

Use `canvas.save()` and `canvas.restore()` when applying a transform or clip that should
not affect subsequent drawing operations. This is cheaper than re-calculating absolute
coordinates for every draw call.

```dart
@override
void paint(Canvas canvas, Size size) {
  // Draw background (unaffected by the transform below)
  canvas.drawRect(Rect.fromLTWH(0, 0, size.width, size.height), _bgPaint);

  // Apply a local transform for the animated element
  canvas.save();
  canvas.translate(size.width / 2, size.height / 2);
  canvas.rotate(animation.value * 2 * math.pi);
  canvas.drawRect(
    Rect.fromCenter(center: Offset.zero, width: 60, height: 60),
    _fillPaint,
  );
  canvas.restore(); // transform no longer in effect

  // Draw overlay (unaffected by the rotation above)
  canvas.drawCircle(Offset(10, 10), 5, _dotPaint);
}
```

Prefer `canvas.saveLayer()` only when you need alpha compositing across multiple draw calls.
`saveLayer` is significantly more expensive because it allocates an offscreen buffer.

## Global Rule 10 Callout

> **Always** wrap `CustomPaint` in `RepaintBoundary`. **Always** pass the animation listenable
> to `repaint:`. **Never** use `setState` to trigger repaints in a `CustomPainter`.

This single rule eliminates the two most common CustomPainter performance bugs: full-tree
rebuilds and unpredictable repaint of static siblings. See SKILL.md for the full code example.
