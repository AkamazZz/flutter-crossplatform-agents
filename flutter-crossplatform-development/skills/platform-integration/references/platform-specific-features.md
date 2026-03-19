---
description: >
  Platform-specific features for iOS, Android, Web, and Desktop with graceful fallback patterns.
---

# Platform-Specific Features

Each target platform exposes capabilities that have no cross-platform Flutter equivalent. The recommended approach is to define an abstract Dart interface and provide a platform-specific implementation, keeping business logic independent of any single platform.

---

## iOS

### Face ID / Touch ID — local_auth

```yaml
dependencies:
  local_auth: ^2.3.0
```

```dart
import 'package:local_auth/local_auth.dart';

final _auth = LocalAuthentication();

Future<bool> authenticateWithBiometrics() async {
  final available = await _auth.canCheckBiometrics;
  if (!available) return false;
  return _auth.authenticate(
    localizedReason: 'Confirm your identity',
    options: const AuthenticationOptions(biometricOnly: true),
  );
}
```

`Info.plist` requires `NSFaceIDUsageDescription`.

### Haptics

```dart
import 'package:flutter/services.dart';

// Light tap feedback
HapticFeedback.lightImpact();

// Selection changed
HapticFeedback.selectionClick();

// Notification (success / warning / error)
HapticFeedback.notificationOccurred(HapticFeedbackType.success); // Flutter 3.x+
```

For richer haptics beyond Flutter's wrapper, invoke via `MethodChannel` calling `UIImpactFeedbackGenerator` or `UINotificationFeedbackGenerator` directly from Swift.

### Live Activities

Live Activities use the ActivityKit framework (iOS 16.1+). Flutter has no built-in support; use a `MethodChannel` to call Swift code that starts, updates, and ends an `Activity<T>`. Pass data as a `Map` through the channel.

### App Clips

An App Clip is a separate Apple target in Xcode. Add an `app_clip` target alongside your main app, share a Flutter engine via `FlutterEngineGroup`, and pass the invocation URL to Dart through a `MethodChannel` on startup.

### WidgetKit (Home Screen Widgets)

Widgets are rendered by a separate Swift extension process and cannot run Dart. Share data with the extension using `App Group` UserDefaults or a shared SQLite file. Update widget content from Flutter via a `MethodChannel` call to `WidgetCenter.shared.reloadAllTimelines()`.

---

## Android

### BiometricPrompt

```yaml
dependencies:
  local_auth: ^2.3.0
```

The same `local_auth` API used for iOS works on Android and uses `BiometricPrompt` internally. `AndroidManifest.xml` requires `USE_BIOMETRIC` permission.

```xml
<uses-permission android:name="android.permission.USE_BIOMETRIC"/>
```

### Home Screen Widgets (AppWidget)

Like iOS WidgetKit, Android AppWidgets run in a separate process. Write the widget in Kotlin using `AppWidgetProvider`, share data via `SharedPreferences` with a known name, and trigger an update from Flutter:

```dart
await const MethodChannel('com.example.app/widget')
    .invokeMethod('updateWidget', {'title': 'New value'});
```

```kotlin
// Kotlin handler
AppWidgetManager.getInstance(context).notifyAppWidgetViewDataChanged(...)
```

### Material You (Dynamic Color)

```yaml
dependencies:
  dynamic_color: ^1.7.0
```

```dart
import 'package:dynamic_color/dynamic_color.dart';

DynamicColorBuilder(
  builder: (ColorScheme? lightDynamic, ColorScheme? lightHighContrast) {
    final scheme = lightDynamic ?? ColorScheme.fromSeed(seedColor: Colors.blue);
    return MaterialApp(theme: ThemeData(colorScheme: scheme));
  },
);
```

### App Links (Deep Links)

Declare intent filters in `AndroidManifest.xml` and verify the domain with a Digital Asset Links JSON file at `/.well-known/assetlinks.json`. Handle the incoming `Uri` in Flutter with `app_links` or `go_router`'s deep-link support.

---

## Web

### PWA Manifest and Service Worker

`web/manifest.json` controls the installable PWA metadata (name, icons, `display: standalone`). Flutter's default build includes a service worker (`flutter_service_worker.js`) that caches assets for offline use. Customize caching strategy by editing `web/index.html`'s service worker registration or replacing the worker entirely.

### JS Interop — dart:js_interop

`dart:js_interop` is the modern, Wasm-compatible API replacing `dart:html` and `package:js`.

```dart
import 'dart:js_interop';

// Declare external JS types
@JS('navigator.clipboard')
external JSObject get clipboard;

extension ClipboardExt on JSObject {
  external JSPromise writeText(JSString text);
}

Future<void> copyToClipboard(String text) async {
  await clipboard.writeText(text.toJS).toDart;
}
```

Use `package:web` for typed bindings to browser DOM and Web APIs instead of writing `@JS` declarations manually.

```yaml
dependencies:
  web: ^1.1.0
```

### Web Rendering

Flutter Web supports two renderers:

- `canvaskit` (`--web-renderer canvaskit`) — pixel-perfect, larger download (~1.5 MB WASM)
- `html`/skwasm (`--web-renderer html`) — smaller, uses browser canvas; some fidelity loss
- `auto` (default) — `canvaskit` on desktop browsers, `html` on mobile

Select at build time: `flutter build web --web-renderer canvaskit`.

---

## Desktop

### System Tray — tray_manager

```yaml
dependencies:
  tray_manager: ^0.2.3
```

```dart
import 'package:tray_manager/tray_manager.dart';

Future<void> initTray() async {
  await trayManager.setIcon('assets/tray_icon.png');
  await trayManager.setContextMenu(Menu(items: [
    MenuItem(key: 'show', label: 'Show Window'),
    MenuItem.separator(),
    MenuItem(key: 'quit', label: 'Quit'),
  ]));
  trayManager.addListener(_TrayListener());
}
```

### Menu Bar (macOS / Linux)

Use `package:flutter/widgets.dart` `PlatformMenuBar` for macOS native menu bar integration:

```dart
PlatformMenuBar(
  menus: [
    PlatformMenu(label: 'File', menus: [
      PlatformMenuItem(
        label: 'New',
        shortcut: const SingleActivator(LogicalKeyboardKey.keyN, meta: true),
        onSelected: () => createNewDocument(),
      ),
    ]),
  ],
  child: myApp,
);
```

### Window Manager — window_manager

```yaml
dependencies:
  window_manager: ^0.4.3
```

```dart
import 'package:window_manager/window_manager.dart';

Future<void> setupWindow() async {
  await windowManager.ensureInitialized();
  await windowManager.setMinimumSize(const Size(400, 300));
  await windowManager.setTitle('My App');
  await windowManager.center();
  await windowManager.show();
}
```

### File System Access

Desktop (and Android/iOS with appropriate permissions) can use `dart:io` directly. For a cross-platform file picker:

```yaml
dependencies:
  file_picker: ^8.1.0
```

```dart
final result = await FilePicker.platform.pickFiles(
  type: FileType.custom,
  allowedExtensions: ['pdf', 'docx'],
);
if (result != null) {
  final file = File(result.files.single.path!);
}
```

### Keyboard Shortcuts

```dart
Shortcuts(
  shortcuts: {
    const SingleActivator(LogicalKeyboardKey.keyS, control: true):
        SaveIntent(),
    const SingleActivator(LogicalKeyboardKey.keyS, control: true, shift: true):
        SaveAsIntent(),
  },
  child: Actions(
    actions: {
      SaveIntent: CallbackAction<SaveIntent>(onInvoke: (_) => save()),
    },
    child: myWidget,
  ),
);
```

---

## Cross-Platform Abstract Interface Pattern

Define the capability in the shared Dart layer; implement it per platform; inject the correct implementation at startup.

```
lib/
  src/
    services/
      biometric_auth_service.dart          // abstract interface
      local_auth_biometric_service.dart    // mobile: local_auth
      web_biometric_service.dart           // web: WebAuthn via dart:js_interop
      unsupported_biometric_service.dart   // desktop stub
```

```dart
// biometric_auth_service.dart
abstract interface class BiometricAuthService {
  Future<bool> isAvailable();
  Future<bool> authenticate({required String reason});
}
```

```dart
// main.dart
BiometricAuthService buildBiometricService() {
  if (kIsWeb) return WebBiometricService();
  if (Platform.isIOS || Platform.isAndroid) return LocalAuthBiometricService();
  return UnsupportedBiometricService();
}

void main() {
  runApp(MyApp(biometricAuth: buildBiometricService()));
}
```

Benefits:
- Business logic and widgets depend only on the interface — fully unit-testable with a fake.
- Adding a new platform means adding one implementation file, not modifying existing code.
- Compile-time errors if a new method is added to the interface but not all implementations.
