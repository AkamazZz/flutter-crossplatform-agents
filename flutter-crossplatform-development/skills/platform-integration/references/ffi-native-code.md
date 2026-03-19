---
description: >
  Dart FFI for C/C++ interop, ffigen binding generation, memory management, and async FFI patterns.
---

# Dart FFI and Native Code

`dart:ffi` lets Dart call functions in shared libraries (`.so`, `.dylib`, `.dll`) that expose a C ABI. Calls are synchronous and bypass the platform channel binary messenger, making FFI well-suited for performance-critical or compute-heavy operations.

---

## DynamicLibrary and NativeFunction

```dart
import 'dart:ffi';
import 'dart:io';

// Load the library at runtime
final DynamicLibrary _lib = Platform.isAndroid
    ? DynamicLibrary.open('libnative.so')
    : DynamicLibrary.process(); // iOS / macOS: statically linked

// C signature:  int32_t add(int32_t a, int32_t b);
typedef _AddNative = Int32 Function(Int32 a, Int32 b);
typedef _AddDart   = int  Function(int  a, int  b);

final _add = _lib.lookupFunction<_AddNative, _AddDart>('add');

void example() {
  print(_add(3, 4)); // 7
}
```

Key points:
- `lookupFunction` takes two type parameters: the native C signature and the Dart equivalent.
- Integer and float types map to `Int8/16/32/64`, `Uint8/16/32/64`, `Float`, `Double`.
- `Void` is used for C `void` return or pointer targets.

---

## ffigen — Automatic Binding Generation

`package:ffigen` reads C headers and generates Dart bindings automatically.

**pubspec.yaml**
```yaml
dev_dependencies:
  ffigen: ^11.0.0

ffigen:
  name: NativeLibrary
  description: Bindings for libnative
  output: lib/src/native_bindings.dart
  headers:
    entry-points:
      - native/include/native.h
  compiler-opts:
    - '-I/usr/local/include'
```

Run generation:
```bash
dart run ffigen
```

The generated `NativeLibrary` class wraps `DynamicLibrary` and exposes all functions as typed Dart methods.

---

## Memory Management

Dart's GC does not manage native memory. Every allocation must be explicitly freed.

**malloc / free with package:ffi**
```dart
import 'package:ffi/ffi.dart';

final ptr = malloc<Int32>(4); // allocate 4 × Int32
try {
  ptr[0] = 10; ptr[1] = 20; ptr[2] = 30; ptr[3] = 40;
  processArray(ptr, 4);
} finally {
  malloc.free(ptr); // always free
}
```

**Finalizer for automatic cleanup**
```dart
final _finalizer = Finalizer<Pointer<Void>>((ptr) => malloc.free(ptr));

class NativeBuffer {
  NativeBuffer(int size) {
    _ptr = malloc<Uint8>(size);
    _finalizer.attach(this, _ptr.cast(), detach: this);
  }

  late final Pointer<Uint8> _ptr;

  void dispose() {
    _finalizer.detach(this);
    malloc.free(_ptr);
  }
}
```

Rules:
- Always `free` inside a `try/finally`, or use `Finalizer` for object-scoped lifetimes.
- Never access a pointer after freeing it.
- `nullptr` comparisons: check `ptr == nullptr` before dereferencing.

---

## Struct Passing

```c
// C header
typedef struct {
  float x;
  float y;
  float z;
} Vec3;

Vec3 normalize(Vec3 v);
```

```dart
// Dart binding
final class Vec3 extends Struct {
  @Float() external double x;
  @Float() external double y;
  @Float() external double z;
}

typedef _NormalizeNative = Vec3 Function(Vec3 v);
typedef _NormalizeDart   = Vec3 Function(Vec3 v);

final _normalize = _lib.lookupFunction<_NormalizeNative, _NormalizeDart>('normalize');

void example() {
  final v = malloc<Vec3>();
  v.ref.x = 1; v.ref.y = 2; v.ref.z = 3;
  final result = _normalize(v.ref);
  malloc.free(v);
  print('${result.x}, ${result.y}, ${result.z}');
}
```

For structs larger than a register, pass and return `Pointer<Vec3>` rather than by value to match the platform ABI.

---

## Callbacks — NativeCallable.listener

`NativeCallable` converts a Dart closure into a C function pointer that native code can store and invoke asynchronously.

```dart
// C signature: void register_callback(void (*cb)(int32_t event_code));

typedef _RegisterCb = Void Function(Pointer<NativeFunction<Void Function(Int32)>>);
final _registerCallback =
    _lib.lookupFunction<_RegisterCb, void Function(Pointer)>('register_callback');

void setupCallback() {
  final callable = NativeCallable<Void Function(Int32)>.listener(
    (int code) {
      print('Native event: $code');
    },
  );
  _registerCallback(callable.nativeFunction);
  // Keep `callable` alive as long as callbacks may arrive.
  // Call callable.close() when done to release resources.
}
```

`NativeCallable.isolateLocal` is for callbacks guaranteed to be invoked on the same isolate (synchronous). `NativeCallable.listener` posts to the event loop and is safe for calls from any thread.

---

## package:ffi Utilities

- `malloc` — allocate / free native memory
- `calloc` — zero-initialised allocation
- `StringUtf8Pointer` extension — `.toNativeUtf8()` / `toDartString()`
- `StringUtf16Pointer` extension — UTF-16 conversion (Windows APIs)
- `Arena` — scoped allocator; frees all on exit

**Arena example**
```dart
import 'package:ffi/ffi.dart';

void withArena() {
  using((Arena arena) {
    final name = 'hello'.toNativeUtf8(allocator: arena);
    final count = arena<Int32>();
    count.value = 5;
    nativeFunction(name, count);
    // arena frees all allocations on scope exit
  });
}
```

---

## Decision Guide: FFI vs Platform Channels

FFI vs Platform Channels:
- Call style — FFI: synchronous; Channels: asynchronous
- Threading — FFI: caller's thread (or native thread for callbacks); Channels: native main thread by default
- Overhead — FFI: minimal (direct C call); Channels: serialization + thread hop
- Access to OS APIs — FFI: no (C ABI only); Channels: yes (full Swift/Kotlin/JS SDK)
- Web support — FFI: no (Wasm interop instead); Channels: partial (JS implementation required)
- Code generation — FFI: `ffigen` from C headers; Channels: manual per platform
- Best for — FFI: C/C++ libraries, compute, SQLite, image processing; Channels: camera, GPS, biometrics, platform UI

Use FFI when you control or have C headers for the native library and need low latency. Use platform channels when you need to call platform SDK APIs that have no C header.
