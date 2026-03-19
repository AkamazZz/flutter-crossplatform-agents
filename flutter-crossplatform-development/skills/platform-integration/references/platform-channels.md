---
description: >
  MethodChannel, EventChannel, BasicMessageChannel patterns with codec selection and error handling.
---

# Platform Channels

Flutter communicates with native code through platform channels. All channel communication is asynchronous and runs over a binary messenger. Messages are serialized with a codec before crossing the Dart/native boundary.

---

## MethodChannel — Request / Response

Use for a single call that returns a result (or throws).

**Dart**
```dart
import 'package:flutter/services.dart';

const _channel = MethodChannel('com.example.app/battery');

Future<int> getBatteryLevel() async {
  try {
    final level = await _channel.invokeMethod<int>('getBatteryLevel');
    return level ?? -1;
  } on PlatformException catch (e) {
    throw BatteryException('Native error: ${e.message}');
  } on MissingPluginException {
    throw BatteryException('Plugin not registered on this platform');
  }
}
```

**Swift (iOS / macOS)**
```swift
// AppDelegate.swift  (or FlutterPlugin registration)
import Flutter

@UIApplicationMain
@objc class AppDelegate: FlutterAppDelegate {
  override func application(
    _ application: UIApplication,
    didFinishLaunchingWithOptions options: [UIApplication.LaunchOptionsKey: Any]?
  ) -> Bool {
    let controller = window?.rootViewController as! FlutterViewController
    let batteryChannel = FlutterMethodChannel(
      name: "com.example.app/battery",
      binaryMessenger: controller.binaryMessenger
    )
    batteryChannel.setMethodCallHandler { call, result in
      guard call.method == "getBatteryLevel" else {
        result(FlutterMethodNotImplemented)
        return
      }
      let device = UIDevice.current
      device.isBatteryMonitoringEnabled = true
      if device.batteryState == .unknown {
        result(FlutterError(code: "UNAVAILABLE", message: "Battery unavailable", details: nil))
      } else {
        result(Int(device.batteryLevel * 100))
      }
    }
    return super.application(application, didFinishLaunchingWithOptions: options)
  }
}
```

**Kotlin (Android)**
```kotlin
// MainActivity.kt
import io.flutter.embedding.android.FlutterActivity
import io.flutter.embedding.engine.FlutterEngine
import io.flutter.plugin.common.MethodChannel

class MainActivity : FlutterActivity() {
  private val channel = "com.example.app/battery"

  override fun configureFlutterEngine(flutterEngine: FlutterEngine) {
    super.configureFlutterEngine(flutterEngine)
    MethodChannel(flutterEngine.dartExecutor.binaryMessenger, channel)
      .setMethodCallHandler { call, result ->
        if (call.method == "getBatteryLevel") {
          val level = getBatteryLevel()
          if (level != -1) result.success(level)
          else result.error("UNAVAILABLE", "Battery unavailable", null)
        } else {
          result.notImplemented()
        }
      }
  }

  private fun getBatteryLevel(): Int {
    val bm = getSystemService(BATTERY_SERVICE) as android.os.BatteryManager
    return bm.getIntProperty(android.os.BatteryManager.BATTERY_PROPERTY_CAPACITY)
  }
}
```

---

## EventChannel — Stream from Native

Use for continuous data: sensors, location, Bluetooth scan results.

**Dart**
```dart
const _channel = EventChannel('com.example.app/accelerometer');

Stream<AccelerometerEvent> get accelerometerStream {
  return _channel
      .receiveBroadcastStream()
      .map((event) => AccelerometerEvent.fromMap(event as Map));
}
```

**Swift**
```swift
let eventChannel = FlutterEventChannel(
  name: "com.example.app/accelerometer",
  binaryMessenger: controller.binaryMessenger
)
eventChannel.setStreamHandler(AccelerometerStreamHandler())

class AccelerometerStreamHandler: NSObject, FlutterStreamHandler {
  private var motionManager: CMMotionManager?

  func onListen(withArguments arguments: Any?, eventSink events: @escaping FlutterEventSink) -> FlutterError? {
    motionManager = CMMotionManager()
    motionManager?.startAccelerometerUpdates(to: .main) { data, error in
      guard let data else { return }
      events(["x": data.acceleration.x, "y": data.acceleration.y, "z": data.acceleration.z])
    }
    return nil
  }

  func onCancel(withArguments arguments: Any?) -> FlutterError? {
    motionManager?.stopAccelerometerUpdates()
    motionManager = nil
    return nil
  }
}
```

**Kotlin**
```kotlin
EventChannel(flutterEngine.dartExecutor.binaryMessenger, "com.example.app/accelerometer")
  .setStreamHandler(object : EventChannel.StreamHandler {
    private var sensorManager: SensorManager? = null
    private var listener: SensorEventListener? = null

    override fun onListen(arguments: Any?, events: EventChannel.EventSink) {
      sensorManager = getSystemService(SENSOR_SERVICE) as SensorManager
      val sensor = sensorManager!!.getDefaultSensor(Sensor.TYPE_ACCELEROMETER)
      listener = object : SensorEventListener {
        override fun onSensorChanged(e: SensorEvent) {
          events.success(mapOf("x" to e.values[0], "y" to e.values[1], "z" to e.values[2]))
        }
        override fun onAccuracyChanged(s: Sensor, a: Int) {}
      }
      sensorManager!!.registerListener(listener, sensor, SensorManager.SENSOR_DELAY_NORMAL)
    }

    override fun onCancel(arguments: Any?) {
      sensorManager?.unregisterListener(listener)
    }
  })
```

---

## BasicMessageChannel — Simple Bidirectional

Use for lightweight, schema-free two-way messaging.

```dart
final _channel = BasicMessageChannel<String>(
  'com.example.app/config',
  StringCodec(),
);

// Send from Dart
await _channel.send('requestConfig');

// Receive messages from native
_channel.setMessageHandler((message) async {
  // handle incoming message from native side
  return 'ack';
});
```

---

## Codec Selection

- `StandardMessageCodec` (default) — supports `null`, `bool`, `int`, `double`, `String`, `Uint8List`, `List`, `Map`; fast binary format, no custom classes
- `JSONMessageCodec` — any JSON-serializable structure; slower, useful for arbitrary nested objects
- `StringCodec` — `String` only; simplest option for plain text
- Custom `MessageCodec<T>` — arbitrary types; implement `encodeMessage` / `decodeMessage`

Prefer `StandardMessageCodec` unless you need types it cannot represent.

---

## Error Handling

```dart
try {
  await channel.invokeMethod('doSomething');
} on PlatformException catch (e) {
  // e.code    — string error code from native
  // e.message — human-readable description
  // e.details — optional extra info (any codec-supported value)
  log('Platform error [${e.code}]: ${e.message}');
} on MissingPluginException {
  // Channel exists in Dart but has no native handler registered.
  // Common in unit tests or when a plugin is not included.
  log('No native implementation available');
}
```

Native side — always call exactly one of `result.success`, `result.error`, or `result.notImplemented`. Calling none or more than one causes a memory leak or crash.

---

## Platform-Specific Implementation Pattern

Structure a plugin so the Dart API is platform-agnostic:

```
lib/
  src/
    battery_platform_interface.dart   // abstract interface
    method_channel_battery.dart       // MethodChannel implementation (default)
  battery.dart                        // public API delegates to interface
android/  ios/  macos/  linux/  windows/  web/
```

```dart
// battery_platform_interface.dart
abstract class BatteryPlatform extends PlatformInterface {
  static BatteryPlatform _instance = MethodChannelBattery();
  static BatteryPlatform get instance => _instance;
  static set instance(BatteryPlatform i) {
    PlatformInterface.verifyToken(i, _token);
    _instance = i;
  }
  static final Object _token = Object();

  Future<int> getBatteryLevel();
}
```

---

## Testing with TestDefaultBinaryMessenger

```dart
import 'package:flutter/services.dart';
import 'package:flutter_test/flutter_test.dart';

void main() {
  TestWidgetsFlutterBinding.ensureInitialized();

  setUp(() {
    TestDefaultBinaryMessengerBinding.instance.defaultBinaryMessenger
        .setMockMethodCallHandler(
      const MethodChannel('com.example.app/battery'),
      (MethodCall call) async {
        if (call.method == 'getBatteryLevel') return 42;
        return null;
      },
    );
  });

  tearDown(() {
    TestDefaultBinaryMessengerBinding.instance.defaultBinaryMessenger
        .setMockMethodCallHandler(
      const MethodChannel('com.example.app/battery'),
      null,
    );
  });

  test('getBatteryLevel returns mocked value', () async {
    expect(await getBatteryLevel(), 42);
  });
}
```

For `EventChannel` streams, set a mock stream handler via `setMockStreamHandler` (available in `flutter_test`).
