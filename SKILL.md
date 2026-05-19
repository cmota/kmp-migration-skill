---
name: kmp-migration
description: >
  Use this skill whenever a user wants to migrate a Kotlin Multiplatform (KMP) or Compose
  Multiplatform project from the old single-module structure (with a `composeApp` module) to
  the new default structure (with a `shared` library module and separate `androidApp`,
  `desktopApp`, `webApp` application modules). Also trigger for questions about the new KMP
  project structure, what changed in the KMP wizard, how to split `composeApp` into separate
  modules, or how to comply with Android Gradle Plugin (AGP) 9.0's requirement to separate
  app and library code. Use this even if the user phrases it casually, e.g. "how do I update
  my KMP project structure", "split my composeApp module", or "AGP 9.0 breaking my KMP build".
---

# KMP Project Structure Migration

Helps users migrate a Kotlin Multiplatform project from the old `composeApp` single-module
structure to the new `shared` + separate app-module structure introduced in May 2026.

## Background: What Changed and Why

The old structure used a single `composeApp` Gradle module that acted as both a KMP library
(holding shared code) and the application entry point for Android, desktop, and web. This
caused several problems:

- Mixed responsibilities made it hard to understand the build configuration
- AGP 9.0 **requires** the Android app entry point to be in a separate module from shared code
- iOS was already in a separate `iosApp` folder — the other platforms were inconsistently co-located
- Amper-based projects already used separate modules per app; Gradle projects were out of step

The new structure separates concerns clearly:

```
project/
├── shared/               ← KMP library: all shared Kotlin/Compose code
│   └── src/
│       ├── commonMain/
│       ├── androidMain/
│       ├── iosMain/
│       ├── desktopMain/
│       └── wasmJsMain/
├── androidApp/           ← Android application module (AGP com.android.application)
├── desktopApp/           ← JVM desktop application module
├── webApp/               ← WASM/JS web application module
├── iosApp/               ← Xcode project (unchanged)
├── gradle/
│   └── libs.versions.toml
└── settings.gradle.kts
```

For variant configurations, see `references/configurations.md`.

---

## Migration Workflow

### Step 1 — Understand the user's project

Before doing anything, ask or infer:

1. **Which targets does the project support?** (Android, iOS, desktop, web/WASM)
2. **Does it have a server module?** If yes, see `references/configurations.md` → "With Server"
3. **Does it use native UI on any platform?** (e.g. SwiftUI for iOS) → see `references/configurations.md` → "Native UI"
4. **What AGP version is in use?** If AGP 9.0+, the migration is mandatory for Android targets.
5. **Do they have the project open or can they share their `build.gradle.kts` files?** Reading the actual files makes the migration much more accurate.

If the user shares files or a directory tree, read them carefully before giving instructions. The
exact package names, dependency declarations, and source set structure all matter.

---

### Step 2 — Create the `shared` module

The `shared` module is a straight extraction of the KMP library portions from `composeApp`.

**What moves to `shared`:**
- All `src/commonMain/`, `src/androidMain/`, `src/iosMain/`, `src/desktopMain/`, `src/wasmJsMain/` source sets
- All shared dependencies (Compose, serialization, coroutines, etc.)
- The `kotlin("multiplatform")` plugin configuration
- `compose.components.resources` resource declarations

**What does NOT move:**
- The `com.android.application` plugin and its configuration (stays in `androidApp`)
- Desktop `application { }` block (stays in `desktopApp`)
- WASM/JS `binaries.executable()` for web app entry (stays in `webApp`)
- App-specific signing configs, package IDs, version codes

**`shared/build.gradle.kts` skeleton:**

```kotlin
plugins {
    alias(libs.plugins.kotlinMultiplatform)
    alias(libs.plugins.composeMultiplatform)
    alias(libs.plugins.composeCompiler)
    alias(libs.plugins.androidLibrary)   // ← library, not application
}

kotlin {
    androidTarget()
    iosX64(); iosArm64(); iosSimulatorArm64()
    jvm("desktop")
    wasmJs { browser() }

    sourceSets {
        commonMain.dependencies {
            implementation(compose.runtime)
            implementation(compose.foundation)
            implementation(compose.material3)
            implementation(compose.components.resources)
            // ... other shared deps
        }
        androidMain.dependencies { /* android-specific shared deps */ }
        val desktopMain by getting { dependencies { /* desktop-specific */ } }
    }
}

android {
    namespace = "com.example.myapp.shared"
    compileSdk = libs.versions.android.compileSdk.get().toInt()
    // minSdk, compileOptions — same as before, but NO applicationId, versionCode, etc.
}
```

---

### Step 3 — Create `androidApp` module

This is a standard Android application module that depends on `shared`.

**`androidApp/build.gradle.kts`:**

```kotlin
plugins {
    alias(libs.plugins.androidApplication)
    alias(libs.plugins.kotlinAndroid)
    alias(libs.plugins.composeCompiler)
}

android {
    namespace = "com.example.myapp"
    compileSdk = libs.versions.android.compileSdk.get().toInt()

    defaultConfig {
        applicationId = "com.example.myapp"
        minSdk = libs.versions.android.minSdk.get().toInt()
        targetSdk = libs.versions.android.targetSdk.get().toInt()
        versionCode = 1
        versionName = "1.0"
    }
    // signing configs, build types, etc.
}

dependencies {
    implementation(projects.shared)
    implementation(libs.androidx.activity.compose)
    // other Android-only deps
}
```

**`androidApp/src/main/`** contains only the Android entry point:
- `AndroidManifest.xml`
- `MainActivity.kt` (calls into shared Compose UI)
- Any Android-specific resources (`res/`)

Move these from `composeApp/src/androidMain/` (or wherever they were). The `MainActivity` body
typically just calls a shared `App()` composable:

```kotlin
class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContent { App() }
    }
}
```

---

### Step 4 — Create `desktopApp` module (if applicable)

**`desktopApp/build.gradle.kts`:**

```kotlin
plugins {
    alias(libs.plugins.kotlinJvm)
    alias(libs.plugins.composeMultiplatform)
    alias(libs.plugins.composeCompiler)
}

dependencies {
    implementation(projects.shared)
    implementation(compose.desktop.currentOs)
}

compose.desktop {
    application {
        mainClass = "com.example.myapp.MainKt"
        // nativeDistributions { ... }
    }
}
```

Move `main()` / `application { Window(...) }` from `composeApp/src/desktopMain/` into
`desktopApp/src/main/kotlin/`.

---

### Step 5 — Create `webApp` module (if applicable)

**`webApp/build.gradle.kts`:**

```kotlin
plugins {
    alias(libs.plugins.kotlinMultiplatform)
    alias(libs.plugins.composeMultiplatform)
    alias(libs.plugins.composeCompiler)
}

kotlin {
    wasmJs {
        moduleName = "webApp"
        browser {
            commonWebpackConfig { outputFileName = "webApp.js" }
        }
        binaries.executable()
    }

    sourceSets {
        commonMain.dependencies {
            implementation(projects.shared)
        }
    }
}
```

Move `index.html`, the WASM entry `main.kt`, and any web-specific resources into `webApp/`.

---

### Step 6 — Update `settings.gradle.kts`

Include all new modules:

```kotlin
include(":shared")
include(":androidApp")
include(":desktopApp")   // if applicable
include(":webApp")       // if applicable
// :iosApp is an Xcode project, not a Gradle module — no change needed
```

Remove the old `:composeApp` include.

---

### Step 7 — iOS: no structural change needed

`iosApp/` stays exactly as it is. It already consumes the shared Kotlin framework via Xcode.
The only thing that may need updating is the framework name in the Xcode build phase if you
renamed the module (from `ComposeApp` to `Shared`):

In Xcode → Build Phases → "Compile Kotlin Framework", update the Gradle task reference:
```
./gradlew :shared:assembleXCFramework
```
And update the `Podfile` or SPM manifest if you use those to reference the new framework name.

---

### Step 8 — Clean up

- Delete the old `composeApp/` directory (after verifying everything builds)
- Update any CI scripts that referenced `:composeApp` tasks
- Update run configurations in IntelliJ IDEA / Android Studio

---

## Common Issues

**"Cannot apply `com.android.application` to a multiplatform module"**
→ This is the AGP 9.0 error. The fix is exactly this migration: move the application plugin
to its own `androidApp` module.

**Duplicate resources / `composeResources` not found**
→ Make sure `compose.components.resources` is only declared in `shared`, and that the
`androidApp`/`desktopApp` modules depend on `projects.shared`, not on a raw JAR.

**`expect`/`actual` declarations not resolved**
→ Confirm the source sets in `shared` still have the same names as before. If you renamed
source sets, update `actual` declarations to match.

**iOS build fails after rename**
→ Check the Xcode scheme's "Compile Kotlin Framework" build phase for the old module name.

---

## References

- `references/configurations.md` — variant structures: native UI, with server module
- Official migration guide: https://kotlinlang.org/docs/multiplatform/multiplatform-project-recommended-structure.html
- AGP 9.0 changes: https://blog.jetbrains.com/kotlin/2026/01/update-your-projects-for-agp9/
- KMP wizard: https://kmp.new
