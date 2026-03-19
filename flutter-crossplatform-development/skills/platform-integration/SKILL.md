---
name: platform-integration
description: >
  Flutter platform channels, FFI, and platform-specific feature implementation. Use when
  integrating native code, building platform channels, using FFI, or implementing
  iOS/Android/Web/Desktop specific features. Trigger on: platform channels, MethodChannel,
  EventChannel, FFI, native code, Swift, Kotlin, JS interop, desktop, web platform.
---

# Platform Integration

## Architecture Overview

```
Flutter (Dart)
      |
      |  MethodChannel / EventChannel / BasicMessageChannel
      |
Platform Channel Layer  (binary messenger, codec, thread dispatch)
      |
      |  Swift (iOS/macOS)  |  Kotlin (Android)  |  JS (Web)  |  C/C++ (Desktop/FFI)
      |
Native Code / OS APIs
```

The platform channel bridge is asynchronous: calls from Dart are serialized with a codec, dispatched to the platform's main thread, handled by a registered handler, and the result is serialized back. FFI bypasses this bridge entirely, enabling synchronous calls directly into compiled native libraries.

## Reference Files

- [platform-channels.md](references/platform-channels.md) — MethodChannel, EventChannel, BasicMessageChannel, codec selection, error handling, testing; use when calling native APIs or streaming native events
- [ffi-native-code.md](references/ffi-native-code.md) — DynamicLibrary, NativeFunction, ffigen, memory management, structs, callbacks; use when binding to C/C++ libraries
- [platform-specific-features.md](references/platform-specific-features.md) — iOS, Android, Web, Desktop features and abstract interface pattern; use when implementing per-platform behavior

## Quick Reference

- `MethodChannel` — one-shot request/response, Dart → Native; battery level, camera capture
- `EventChannel` — continuous stream from native to Dart; accelerometer, GPS, Bluetooth scan
- `BasicMessageChannel` — simple bidirectional messaging; lightweight key-value exchange
- `dart:ffi` — synchronous calls into C/C++ libraries; image processing, crypto, SQLite
- `dart:js_interop` — browser JS API access; WebRTC, Web Storage, browser APIs

## Anti-Patterns

**Calling platform channels on background isolates without setup.**
`MethodChannel` requires the root isolate's binary messenger by default. Use `BackgroundIsolateBinaryMessenger.ensureInitialized(token)` before invoking channels from a background isolate, or route calls through the main isolate.

**Blocking the platform main thread.**
Channel handlers on iOS/Android run on the main thread. Offload heavy work with `DispatchQueue.global()` (Swift) or `coroutineScope` / `Executors` (Kotlin) and reply on the main thread.

**Skipping codec selection.**
`StandardMessageCodec` supports a limited type set (primitives, lists, maps, `Uint8List`). Passing unsupported types silently fails or throws at runtime. Use `JSONMessageCodec` for arbitrary nested objects or write a custom `MessageCodec`.

**Leaking native memory with FFI.**
`malloc`-allocated memory is not garbage-collected. Always pair allocations with `free` or register a `Finalizer`. Forgetting this causes unbounded native heap growth.

**Mixing platform-specific code into shared widgets.**
`Platform.isIOS` checks scattered across widget trees create untestable, fragile code. Use the abstract-interface pattern from `platform-specific-features.md` to isolate platform decisions.

**Using `dart:html` for new Web code.**
`dart:html` is legacy and incompatible with Wasm compilation. Use `dart:js_interop` and `package:web` for all new web platform interop.
