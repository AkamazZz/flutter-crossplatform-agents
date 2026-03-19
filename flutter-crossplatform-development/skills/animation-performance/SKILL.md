---
name: animation-performance
description: >
  Flutter animation and rendering performance with AnimationController, CustomPainter, shaders,
  and RepaintBoundary. Use when implementing animations, CustomPainter, diagnosing jank, or
  optimizing rendering. Trigger on: AnimationController, CustomPainter, Tween, RepaintBoundary,
  shaders, RenderObject, frame rates, performance, animation, Impeller, jank.
---

# Flutter Animation & Performance Skill

This skill enforces animation and rendering performance standards for Flutter. Before answering,
read the relevant reference file(s) for the topic at hand.

## Architecture Overview

```
Ticker (vsync) → AnimationController → CurvedAnimation/Tween → AnimatedBuilder/Widget/CustomPainter
```

The animation pipeline flows from a `Ticker` (provided by `SingleTickerProviderStateMixin` or
a custom `Ticker`) through an `AnimationController` that drives time. A `CurvedAnimation`
applies easing, and a `Tween` maps the 0–1 range to typed values. The animated value reaches
the UI via `AnimatedBuilder` (widget subtree) or directly into a `CustomPainter` via the
`repaint:` parameter — bypassing the build phase entirely.

## Reference Files — Read Before Answering

| Topic | Reference file | When to read |
|---|---|---|
| AnimationController, Tweens, implicit/explicit animations | `references/animation-controllers.md` | Setting up controllers, CurvedAnimation, staggered/Hero animations, Rive/Lottie |
| CustomPainter, RepaintBoundary, canvas batching | `references/custom-painter.md` | Writing CustomPainter, canvas optimization, repaint isolation |
| Shaders, RenderObject, custom Tickers | `references/shaders-render-objects.md` | Fragment shaders, LeafRenderObjectWidget, markNeedsPaint, custom Tickers |
| Full performance checklist | `references/performance-checklist.md` | Code review, diagnosing jank, pre-merge checklist |

**Rule**: Always read the relevant reference(s) in full before writing or reviewing animation code.
Multiple references may apply — e.g. a shader-driven CustomPainter needs both
`custom-painter.md` and `shaders-render-objects.md`.

## Quick Reference

| Concern | Rule |
|---|---|
| FPS / refresh rate | Let Flutter manage it — never use `Timer.periodic` for animation |
| Rebuilds | `AnimatedBuilder` over `setState` for animation listeners |
| Repaint isolation | Wrap animated content in `RepaintBoundary` |
| Widget structure | Never change tree structure during an active animation |
| CustomPainter | Pass `AnimationController` to `repaint:` param — never `setState` |
| Paint objects | Define `Paint` once as a field — never instantiate inside `paint()` |
| Draw calls | Batch with `drawPoints` / `drawAtlas` |
| Shaders | Fragment shaders for complex per-pixel GPU effects |
| Tickers | Custom `Ticker` for precise frame control with lower overhead |
| RenderObject updates | `markNeedsPaint()` — never `setState` |

## Common Pitfalls

1. **`setState` inside an animation listener** — rebuilds the entire widget tree every frame; use `AnimatedBuilder` or the `repaint:` parameter instead.
2. **`new Paint()` inside `paint()`** — allocates a new object every frame at 60–120 Hz; define `Paint` as a class field and reuse it.
3. **Changing widget tree structure mid-animation** — adding, removing, or reordering nodes triggers a full layout pass and causes jank; toggle visibility via `Opacity` instead.
4. **`CustomPainter` driven by `setState`** — triggers build + paint phases; pass the `Listenable` to `repaint:` to trigger paint phase only.
5. **Manual `Timer.periodic` for frame loop** — does not synchronize with vsync, causes tearing and wasted frames; always use `AnimationController` or a custom `Ticker`.
