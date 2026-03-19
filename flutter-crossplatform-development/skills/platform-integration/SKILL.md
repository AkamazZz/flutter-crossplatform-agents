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

| File | Contents | Use When |
|---|---|---|
| [platform-channels.md](references/platform-channels.md) | MethodChannel, EventChannel, BasicMessageChannel, codec selection, error handling, testing | Calling native APIs or streaming native events |
| [ffi-native-code.md](references/ffi-native-code.md) | DynamicLibrary, NativeFunction, ffigen, memory management, structs, callbacks | Binding to C/C++ libraries or performance-critical native code |
| [platform-specific-features.md](references/platform-specific-features.md) | iOS, Android, Web, Desktop platform features; abstract interface pattern | Implementing features that differ per target platform |

## Quick Reference

| Channel Type | Use Case | Direction | Example |
|---|---|---|---|
| `MethodChannel` | One-shot request/response | Dart → Native (+ response) | Battery level, camera capture |
| `EventChannel` | Continuous stream from native | Native → Dart (stream) | Accelerometer, GPS, Bluetooth scan |
| `BasicMessageChannel` | Simple bidirectional messaging | Both directions | Lightweight key-value exchange |
| `dart:ffi` | Synchronous native library calls | Dart ↔ C ABI | Image processing, crypto, SQLite |
| `dart:js_interop` | Browser JS API access | Dart ↔ JavaScript | WebRTC, Web Storage, browser APIs |

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
