# ShoppingGate SDK Integration Guide

This documentation provides comprehensive instructions for integrating the **ShoppingGate SDK** into your Flutter project. It covers both Single Module and Multiple Module architectures for Android Native integration.

---

## 📖 Table of Contents
1. [Single Module Integration](#1-single-module-integration)
2. [Multiple Module Integration (Bridge Approach)](#2-multiple-module-integration-bridge-approach)
    - [Phase 1: Flutter Bridge Setup](#phase-1-flutter-bridge-setup)
    - [Phase 2: Dependency Configuration](#phase-2-dependency-configuration)
    - [Phase 3: Entry Point Implementation](#phase-3-entry-point-implementation)
    - [Phase 4: Building and Exporting AAR](#phase-4-building-and-exporting-aar)
    - [Phase 5: Android Native Configuration](#phase-5-android-native-configuration)
    - [Phase 6: Launching Modules from Native](#phase-6-launching-modules-from-native)

---

## 1. Single Module Integration
For standard single-module implementations, please refer to the official documentation link below:
🔗 [ShoppingGate SDK Single Module Docs](https://vasudev-rathod-eww.github.io/shoppinggate_sdk_flutter/)

---

## 2. Multiple Module Integration (Bridge Approach)

Follow these steps to integrate multiple Flutter modules (e.g., ShoppingGate and a custom Module A) into a single Android Native host application.

### Phase 1: Flutter Bridge Setup
Create a new Flutter module that will act as a bridge between your native app and multiple sub-modules.

```bash
# Step 1: Create the bridge module
flutter create -t module bridge_module
```

### Phase 2: Dependency Configuration
In your `bridge_module/pubspec.yaml`, add the ShoppingGate SDK and your internal modules.

```yaml
# Step 2: Add dependencies
dependencies:
  shoppingggate_flutter_sdk:
    git:
      url: https://@github.com/vasudev-rathod-eww/shoppingate_sdk.git
      ref: main # or a specific tag/branch
      path: shoppingggate_flutter_sdk # main repo folder name
  module_a:
    path: ../module_a
```

**Step 3: Install Dependencies**
Run the following command in your terminal:
```bash
flutter pub get
```
> **Note:** If prompted for a password or token, use: `iowebfiuwciweo3833092be2`

### Phase 3: Entry Point Implementation
Modify `lib/main.dart` in your **BridgeModule** to define specific entry points for each module using `@pragma('vm:entry-point')`.

```dart
// Step 4: main.dart implementation
import 'package:flutter/material.dart';
import 'package:shoppingggate_flutter_sdk/main.dart' as shoppingateModule;
import 'package:module_a/main.dart' as moduleA;

void main() async {
  try {
    WidgetsFlutterBinding.ensureInitialized();
    await shoppingateModule.main();
  } catch (e, stacktrace) {
    print("DEBUG: Main ERROR IN DART: $e");
  }
}

// --- Entry point for ShoppingGate ---
@pragma('vm:entry-point')
Future<void> launchShoppingGate() async {
  try {
    WidgetsFlutterBinding.ensureInitialized();
    await shoppingateModule.main();
  } catch (e, stacktrace) {
    print("DEBUG: Main ERROR IN DART: $e");
  }
}

// --- Entry point for Module A ---
@pragma('vm:entry-point')
void launchAModule() {
  WidgetsFlutterBinding.ensureInitialized();
  moduleA.main();
}
```

### Phase 4: Building and Exporting AAR
**Step 5:** Generate the Android Archive (AAR) files.
```bash
flutter build aar --no-debug --no-profile
```

**Step 6 & 7:**
1. Navigate to `root > build > host > outputs > repo`.
2. Copy the `repo` folder.
3. Paste it into your **Android Native root folder** and rename it to `BridgeModule`.

### Phase 5: Android Native Configuration

**Step 8: Update `settings.gradle.kts`**
Add the local repository path and Flutter engine binaries.

```kotlin
dependencyResolutionManagement {
    repositories {
        // ... other repositories
        
        // Local ShoppingGate SDK Repository
        maven {
            url = uri("${rootProject.projectDir}/BridgeModule") 
        }

        // Flutter engine binaries
        maven { 
            url = uri("https://storage.googleapis.com/download.flutter.io") 
        }
    }
}
```

**Step 9: Update `app/build.gradle.kts`**
Configure NDK filters, Java 17 compatibility, and dependencies.

```kotlin
android {
    defaultConfig {
        // Filter for architectures supported by Flutter
        ndk { 
            abiFilters += listOf("armeabi-v7a", "arm64-v8a", "x86_64") 
        }
        multiDexEnabled = true
    }
    
    compileOptions {
        isCoreLibraryDesugaringEnabled = true 
        sourceCompatibility = JavaVersion.VERSION_17
        targetCompatibility = JavaVersion.VERSION_17
    }
    
    kotlin {
        compilerOptions {
            jvmTarget.set(JvmTarget.JVM_17)
        }
    }
}

tasks.withType<org.jetbrains.kotlin.gradle.tasks.KotlinCompile>().configureEach {
    compilerOptions {
        jvmTarget.set(JvmTarget.JVM_17)
    }
}

dependencies {
    // Replace 'androidPackage' with the actual package name found in BridgeModule's pubspec.yaml
    implementation("androidPackage:flutter_release:1.0")
    coreLibraryDesugaring("com.android.tools:desugar_jdk_libs:2.1.5")
}
```

**Step 10: Permissions**
Ensure all permissions required by the ShoppingGate SDK (as documented in the [Single Module link](#1-single-module-integration)) are added to your `AndroidManifest.xml`.

### Phase 6: Launching Modules from Native

**Step 11: Implement Activity Logic**
Use the following Kotlin code to initialize the Flutter Engines and handle the MethodChannel for data passing.

```kotlin
import io.flutter.embedding.android.FlutterActivity
import io.flutter.embedding.engine.FlutterEngine
import io.flutter.embedding.engine.FlutterEngineCache
import io.flutter.embedding.engine.dart.DartExecutor
import io.flutter.plugin.common.MethodChannel
import io.flutter.plugins.GeneratedPluginRegistrant
import io.flutter.FlutterInjector

class MainActivity : ComponentActivity() {
    private var flutterEngineShoppingGate: FlutterEngine? = null
    private var flutterEngineTest: FlutterEngine? = null

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContent {
            Column {
                // Button to Launch ShoppingGate SDK
                Button(onClick = {
                    FlutterEngineCache.getInstance().remove("ShoppingGateModule")
                    flutterEngineShoppingGate?.destroy()
                    flutterEngineShoppingGate = FlutterEngine(this@MainActivity)
                    
                    flutterEngineShoppingGate?.let { engine ->
                        GeneratedPluginRegistrant.registerWith(engine)

                        MethodChannel(engine.dartExecutor.binaryMessenger, "shoppingate_sdk/channel")
                            .setMethodCallHandler { call, result ->
                                if (call.method == "getInitialData") {
                                    val data = mapOf(
                                        "mobile" to "555555555",
                                        "environment" to "development",
                                        "language" to "en"
                                    )
                                    result.success(data)
                                } else {
                                    result.notImplemented()
                                }
                            }

                        val loader = FlutterInjector.instance().flutterLoader()
                        loader.startInitialization(this@MainActivity)
                        loader.ensureInitializationComplete(this@MainActivity, null)
                        
                        val entrypoint = DartExecutor.DartEntrypoint(
                            loader.findAppBundlePath(),
                            "launchShoppingGate" // Matches Dart function name
                        )
                        engine.dartExecutor.executeDartEntrypoint(entrypoint)
                        FlutterEngineCache.getInstance().put("ShoppingGateModule", engine)

                        startActivity(
                            FlutterActivity.withCachedEngine("ShoppingGateModule")
                                .backgroundMode(FlutterActivityLaunchConfigs.BackgroundMode.opaque)
                                .build(this@MainActivity)
                        )
                    }
                }) {
                    Text("Launch ShoppingGate")
                }

                Spacer(modifier = Modifier.height(30.dp))

                // Button to Launch Test Module (Module A)
                Button(onClick = {
                    FlutterEngineCache.getInstance().remove("AModule")
                    flutterEngineTest?.destroy()
                    flutterEngineTest = FlutterEngine(this@MainActivity)
                    
                    flutterEngineTest?.let { engine ->
                        GeneratedPluginRegistrant.registerWith(engine)
                        val loader = FlutterInjector.instance().flutterLoader()
                        loader.startInitialization(this@MainActivity)
                        loader.ensureInitializationComplete(this@MainActivity, null)
                        
                        val entrypoint = DartExecutor.DartEntrypoint(
                            loader.findAppBundlePath(),
                            "launchAModule" // Matches Dart function name
                        )
                        engine.dartExecutor.executeDartEntrypoint(entrypoint)
                        FlutterEngineCache.getInstance().put("AModule", engine)

                        startActivity(
                            FlutterActivity.withCachedEngine("AModule")
                                .backgroundMode(FlutterActivityLaunchConfigs.BackgroundMode.opaque)
                                .build(this@MainActivity)
                        )
                    }
                }) {
                    Text("Launch TestModule")
                }
            }
        }
    }
}
```

**Step 12: Run the Project**
Build and run your Android application. The bridge module will now correctly route the requests to either the ShoppingGate SDK or Module A based on the triggered entry point.
