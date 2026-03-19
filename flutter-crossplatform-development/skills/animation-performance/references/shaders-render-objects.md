---
description: Custom Tickers, RenderObject with markNeedsPaint, fragment shaders, LeafRenderObjectWidget, decision guide
---

# Shaders & RenderObjects Reference

## Custom Ticker for Precise Frame Control

`AnimationController` is convenient but has overhead from the Tween/Animation pipeline. When
you need lower-level frame timing — for instance, driving a custom physics simulation or a
shader that needs exact elapsed time — use a `Ticker` directly.

**Rule**: Use a raw `Ticker` (via `SingleTickerProviderStateMixin.createTicker`) when you need
elapsed-time values or sub-frame precision without the Tween/controller abstraction.

```dart
class _PhysicsState extends State<PhysicsWidget>
    with SingleTickerProviderStateMixin {
  late final Ticker _ticker;
  double _value = 0.0;

  @override
  void initState() {
    super.initState();
    _ticker = createTicker(_onTick)..start();
  }

  void _onTick(Duration elapsed) {
    final seconds = elapsed.inMilliseconds / 1000.0;
    _value = (seconds % 2.0) / 2.0; // 0→1 repeating over 2 s
    // Notify the render object directly — see section below
    _renderKey.currentContext
        ?.findRenderObject()
        ?.markNeedsPaint();
  }

  @override
  void dispose() {
    _ticker.dispose(); // ✅ Always dispose
    super.dispose();
  }
}
```

Key points:
- The `elapsed` duration is relative to when `start()` was called — no drift.
- A `Ticker` that is not stopped when the widget is off-screen still fires. Use `ticker.muted`
  or stop/start around `AppLifecycleState` changes if the animation is expensive.
- Always dispose the `Ticker` in `dispose()`.

## RenderObject with markNeedsPaint — Bypass the Widget Tree

For the highest-performance animations, bypass the Widget and Element trees entirely. A custom
`RenderObject` registers as a listener on an `Animation` and calls `markNeedsPaint()` directly,
scheduling only a repaint — no build, no layout.

**Rule**: Use `markNeedsPaint()` (not `setState`) when a `RenderObject` needs to redraw due to
an animation value change.

### LeafRenderObjectWidget Pattern

```dart
/// Widget side — thin shell, no state of its own
class AnimatedRenderWidget extends LeafRenderObjectWidget {
  final Animation<double> animation;

  const AnimatedRenderWidget({required this.animation, super.key});

  @override
  RenderObject createRenderObject(BuildContext context) =>
      RenderAnimatedBox(animation);

  @override
  void updateRenderObject(
    BuildContext context,
    RenderAnimatedBox renderObject,
  ) {
    renderObject.animation = animation; // setter handles listener swap
  }
}

/// RenderObject side — owns painting, driven by animation listener
class RenderAnimatedBox extends RenderBox {
  RenderAnimatedBox(Animation<double> animation) : _animation = animation {
    _animation.addListener(markNeedsPaint);
  }

  Animation<double> _animation;

  Animation<double> get animation => _animation;

  set animation(Animation<double> value) {
    if (_animation == value) return;
    _animation.removeListener(markNeedsPaint);
    _animation = value;
    _animation.addListener(markNeedsPaint);
    markNeedsPaint();
  }

  @override
  bool get sizedByParent => true;

  @override
  Size computeDryLayout(BoxConstraints constraints) => constraints.biggest;

  @override
  void paint(PaintingContext context, Offset offset) {
    final canvas = context.canvas;
    final center = offset + Offset(size.width / 2, size.height / 2);
    final radius = 50.0 * _animation.value;
    final paint = Paint()
      ..color = Color.lerp(Colors.red, Colors.blue, _animation.value)!;
    canvas.drawCircle(center, radius, paint);
  }

  @override
  void detach() {
    _animation.removeListener(markNeedsPaint); // ✅ Clean up listener
    super.detach();
  }
}
```

Important details:
- Remove the listener in `detach()` to prevent memory leaks after the widget leaves the tree.
- If layout is also animation-driven, call `markNeedsLayout()` instead (more expensive).
- `PaintingContext.canvas` should only be used within a single `paint()` call; do not cache it.

## Fragment Shaders — GPU-Accelerated Pixel Effects

Fragment shaders run on the GPU and are the correct tool for per-pixel effects: gradients,
noise, distortions, colour grading, and other operations that would be prohibitively expensive
in Dart.

### Setup

1. Write the shader in GLSL (Flutter uses a subset — no `#version` directive, use `#include
   <flutter/runtime_effect.glsl>`).
2. Declare the asset in `pubspec.yaml`:

```yaml
flutter:
  shaders:
    - shaders/my_effect.frag
```

3. Load the shader once (e.g., in `initState` or a `FutureBuilder`) and cache the
   `FragmentShader` instance. Never load it inside `paint()`.

### GLSL Shader Example

```glsl
// shaders/wave.frag
#include <flutter/runtime_effect.glsl>

uniform float uWidth;
uniform float uHeight;
uniform float uTime;

out vec4 fragColor;

void main() {
  vec2 uv = FlutterFragCoord().xy / vec2(uWidth, uHeight);
  float wave = sin(uv.x * 10.0 + uTime * 3.0) * 0.5 + 0.5;
  fragColor = vec4(uv.x, wave, uv.y, 1.0);
}
```

### Dart Integration

```dart
class ShaderScreen extends StatefulWidget {
  const ShaderScreen({super.key});

  @override
  State<ShaderScreen> createState() => _ShaderScreenState();
}

class _ShaderScreenState extends State<ShaderScreen>
    with SingleTickerProviderStateMixin {
  ui.FragmentShader? _shader;
  late final AnimationController _controller;

  @override
  void initState() {
    super.initState();
    _controller = AnimationController(
      vsync: this,
      duration: const Duration(seconds: 10),
    )..repeat();
    _loadShader();
  }

  Future<void> _loadShader() async {
    final program =
        await ui.FragmentProgram.fromAsset('shaders/wave.frag');
    if (mounted) {
      setState(() => _shader = program.fragmentShader());
    }
  }

  @override
  Widget build(BuildContext context) {
    final shader = _shader;
    if (shader == null) return const SizedBox.shrink();

    return RepaintBoundary(
      child: CustomPaint(
        painter: ShaderPainter(shader: shader, animation: _controller),
        size: Size.infinite,
      ),
    );
  }

  @override
  void dispose() {
    _controller.dispose();
    super.dispose();
  }
}

class ShaderPainter extends CustomPainter {
  ShaderPainter({required this.shader, required Animation<double> animation})
      : super(repaint: animation),
        _animation = animation;

  final ui.FragmentShader shader;
  final Animation<double> _animation;

  @override
  void paint(Canvas canvas, Size size) {
    // Set uniforms — order must match the shader's uniform declarations
    shader.setFloat(0, size.width);
    shader.setFloat(1, size.height);
    shader.setFloat(2, _animation.value * 10.0); // uTime (0–10)

    canvas.drawRect(
      Rect.fromLTWH(0, 0, size.width, size.height),
      Paint()..shader = shader,
    );
  }

  @override
  bool shouldRepaint(ShaderPainter old) =>
      old._animation != _animation || old.shader != shader;
}
```

### Shader Uniforms

- `uniform float` → `shader.setFloat(index, value)`
- `uniform vec2` → `shader.setFloat(i, x); shader.setFloat(i+1, y)`
- `uniform sampler2D` → `shader.setImageSampler(index, image)`

Uniforms are indexed in declaration order. A `vec2` occupies two consecutive float slots.

## Decision Guide — Which Technique to Use

- Simple property animation (size, colour, opacity) → implicit widget (`AnimatedContainer`, `AnimatedOpacity`)
- Multi-property or sequenced animation → `AnimationController` + `Tween` + `AnimatedBuilder`
- Custom drawn animation (shapes, paths, particles) → `CustomPainter` with `repaint:` parameter
- Many particles or sprites per frame → `CustomPainter` + `drawAtlas` / `drawPoints`
- Per-pixel GPU effect (noise, distortion, colour grade) → Fragment shader + `CustomPainter`
- Precise frame timing / physics simulation → Custom `Ticker`
- High-frequency animation bypassing build phase → `LeafRenderObjectWidget` + `markNeedsPaint()`
- Character or motion-graphic animation → Rive (state machines) or Lottie (JSON playback)
- Shared element between routes → `Hero` widget

**General escalation order**: implicit → explicit → CustomPainter → RenderObject/shader.
Reach for lower levels only when profiling confirms the higher-level approach is the
bottleneck.
