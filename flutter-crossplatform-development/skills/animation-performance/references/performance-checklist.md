---
description: Full animation and rendering performance checklist — use during code review, pre-merge, and jank diagnosis
---

# Animation & Rendering Performance Checklist

Use this checklist during code review and when diagnosing frame drops. Items are grouped by
concern. Each item is marked with a severity rating when it represents an anti-pattern.

Severity scale:
- **[CRITICAL]** — causes measurable jank or memory leaks in most cases
- **[HIGH]** — likely performance regression under normal load
- **[MEDIUM]** — degrades performance under stress (many widgets, low-end devices)
- **[LOW]** — style/maintainability concern with minor perf implications

---

## RepaintBoundary Isolation

- ✅ Every `CustomPaint` that animates is wrapped in a `RepaintBoundary`
- ✅ Animated widgets that live next to large static subtrees are wrapped in `RepaintBoundary`
- ✅ `RepaintBoundary` is placed as close to the animated content as possible (not at the root)
- ❌ **[HIGH]** `RepaintBoundary` added to every widget indiscriminately — each boundary allocates a GPU texture; over-use wastes VRAM
- ❌ **[MEDIUM]** Animated content has no `RepaintBoundary` — static siblings are re-rasterized on every frame

---

## Animation Setup

- ✅ `AnimationController` is created in `initState`, not in `build`
- ✅ `AnimationController.dispose()` is called in `State.dispose()`
- ✅ Custom `Ticker`s are disposed in `State.dispose()`
- ✅ `vsync: this` is passed to every `AnimationController` (requires `TickerProviderStateMixin`)
- ✅ `SingleTickerProviderStateMixin` for one controller; `TickerProviderStateMixin` for multiple
- ✅ Animation duration and curve are `const` where possible
- ❌ **[CRITICAL]** `AnimationController` not disposed — leaks vsync binding, causes "disposed with active Ticker" exceptions
- ❌ **[CRITICAL]** `Timer.periodic` used instead of `AnimationController` — not vsync-synchronized, causes tearing and wasted CPU cycles
- ❌ **[HIGH]** `AnimationController` created inside `build` — new controller on every rebuild, never disposed
- ❌ **[HIGH]** Ticker not disposed — continues firing after widget is removed from tree

---

## CustomPainter

- ✅ `AnimationController` (or `Listenable`) is passed to `repaint:` parameter of `CustomPainter`
- ✅ `shouldRepaint` returns `true` only when the painter's inputs have actually changed
- ✅ `shouldRepaint` is implemented (not left as `return true` always)
- ✅ `CustomPaint` is wrapped in `RepaintBoundary`
- ✅ `Paint` objects are defined as class fields, not created inside `paint()`
- ✅ Multiple shapes/sprites are batched with `drawPoints` or `drawAtlas`
- ✅ `canvas.save()` / `canvas.restore()` brackets any transform or clip that should not affect subsequent draws
- ✅ `canvas.saveLayer()` is only used when alpha compositing across draw calls is required
- ❌ **[CRITICAL]** `setState` used to drive `CustomPainter` — rebuilds entire widget tree at animation frame rate
- ❌ **[HIGH]** `new Paint()` inside `paint()` method — heap allocation every frame (60–120 Hz)
- ❌ **[HIGH]** `shouldRepaint` always returns `true` without checking inputs — forces repaint even when nothing changed
- ❌ **[MEDIUM]** Per-item `drawCircle`/`drawRect` in a loop instead of batched operations — excessive GPU draw calls
- ❌ **[MEDIUM]** `canvas.saveLayer()` used for simple transforms — allocates offscreen buffer unnecessarily

---

## Widget Rebuild Optimisation

- ✅ `AnimatedBuilder` is used instead of `setState` for animation-driven rebuilds
- ✅ The `child` argument of `AnimatedBuilder` is used for stable, non-animated sub-trees
- ✅ `const` constructors are used for widgets that do not depend on animation values
- ✅ `buildWhen` is used on `BlocBuilder` to prevent animation-frame rebuilds from unrelated state changes
- ✅ Widget tree structure does not change during an active animation (no conditional node insertion/removal)
- ✅ Animated content uses `Opacity`/`FadeTransition` rather than conditional tree mutations for show/hide
- ❌ **[HIGH]** Entire screen widget rebuilt inside `AnimatedBuilder.builder` — builder should be scoped to the animated region
- ❌ **[HIGH]** Widget tree nodes conditionally added or removed during animation — triggers full layout pass, causes jank
- ❌ **[MEDIUM]** Expensive widgets inside `AnimatedBuilder.builder` that could be hoisted to the `child` argument
- ❌ **[LOW]** Missing `const` on widgets that never change — prevents build-time optimisation

---

## RenderObject Updates

- ✅ `markNeedsPaint()` is called instead of `setState` when a `RenderObject` needs to redraw
- ✅ Animation listener is removed in `RenderObject.detach()` to prevent memory leaks
- ✅ `markNeedsLayout()` is only called when the render object's size actually changes
- ✅ `LeafRenderObjectWidget.updateRenderObject` updates the render object's properties without creating a new instance
- ❌ **[CRITICAL]** Animation listener added in `RenderObject` but not removed in `detach()` — memory leak; listener fires after disposal
- ❌ **[HIGH]** `markNeedsLayout()` called when only painting changed — layout is more expensive than paint; use `markNeedsPaint()` when size is unchanged

---

## Fragment Shaders

- ✅ `FragmentProgram.fromAsset` is called once (in `initState` or via a `FutureBuilder`), not inside `paint()`
- ✅ Shader asset path is declared under `flutter: shaders:` in `pubspec.yaml`
- ✅ Uniforms are set in the correct order (matching GLSL declaration order)
- ✅ `ShaderPainter` uses `repaint:` parameter, not `setState`
- ✅ `FragmentShader` instance is reused across frames; only uniform values change each frame
- ❌ **[CRITICAL]** `FragmentProgram.fromAsset` called inside `paint()` — async load on every frame, causes exceptions and severe jank
- ❌ **[HIGH]** Shader asset not listed in `pubspec.yaml` — runtime exception when loading
- ❌ **[MEDIUM]** Uniforms set in wrong order — incorrect rendering with no compile-time error

---

## Performance Monitoring Workflow

### Enable the Performance Overlay

```dart
MaterialApp(
  showPerformanceOverlay: true, // Top bar: GPU thread, bottom bar: UI (Dart) thread
)
```

Both bars should stay well below the target frame budget line (16.7 ms for 60 Hz,
8.3 ms for 120 Hz). A red bar indicates a dropped frame.

### Timeline API — Instrument Specific Sections

```dart
import 'dart:developer' as developer;

void _onTick(Duration elapsed) {
  developer.Timeline.startSync('MyAnimation.paint');
  // ... animation work ...
  developer.Timeline.finishSync();
}
```

View the timeline in Flutter DevTools → Performance → Timeline Events.

### Flutter DevTools Profiling Workflow

1. Run the app in profile mode: `flutter run --profile`
2. Open DevTools: `flutter devtools` or via IDE
3. Navigate to **Performance** tab
4. Record a session while the animation plays
5. Look for:
   - **UI thread** (Dart): long frames signal expensive build/layout/paint Dart code
   - **Raster thread** (GPU): long frames signal expensive shader compilation or draw calls
   - **Widget Rebuilds** panel: identify which widgets rebuild most frequently
6. Navigate to **Inspector** → **Highlight Repaints** to visualise which layers repaint each frame (flashing colour indicates repaint)
7. Use **Memory** tab to detect allocations growing in step with animation frames — sign of Paint objects or collections created inside `paint()`

### Impeller (Flutter's Rendering Engine)

Impeller pre-compiles shaders at app startup, eliminating the shader-compilation jank seen
with the legacy Skia backend. It is the default on iOS and opt-in on Android.

To enable on Android for testing:
```yaml
# android/app/src/main/AndroidManifest.xml
<meta-data android:name="io.flutter.embedding.android.EnableImpeller" android:value="true" />
```

With Impeller, first-frame jank from shader compilation is eliminated, but complex
`saveLayer` calls and certain path operations may be slower than Skia on some hardware.
Profile on both backends if targeting Android broadly.

---

## Common Pitfalls — Severity Summary

- `setState` inside animation listener — **CRITICAL** — use `AnimatedBuilder` or `repaint:` param
- `AnimationController` not disposed — **CRITICAL** — `dispose()` in `State.dispose()`
- `new Paint()` inside `paint()` — **HIGH** — define `Paint` as class field
- `Timer.periodic` instead of controller — **CRITICAL** — use `AnimationController` with `vsync:`
- Widget tree mutation during animation — **HIGH** — use `Opacity`/`FadeTransition` instead
- `shouldRepaint` always returns `true` — **HIGH** — compare old/new inputs
- No `RepaintBoundary` around CustomPaint — **MEDIUM** — wrap with `RepaintBoundary`
- Per-item draw calls in loop — **MEDIUM** — batch with `drawPoints`/`drawAtlas`
- `saveLayer` for simple transforms — **MEDIUM** — use `save`/`restore` instead
- RenderObject listener not removed in `detach` — **CRITICAL** — remove in `detach()` override
- Shader loaded inside `paint()` — **CRITICAL** — load once in `initState`/`FutureBuilder`
- Animating `Padding`/`Container` size — **HIGH** — use `Transform` or `SizedBox` transitions
