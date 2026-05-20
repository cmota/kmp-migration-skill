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

The new structure separates concerns clearly. For variant configurations (native UI, server), see `references/configurations.md`.

---

## Migration Workflow

### Step 1 — Understand the user's project

Before doing anything, ask or infer:

1. **Which targets does the project support?** (Android, iOS, desktop, web/WASM)
2. **Does it have a server module?** If yes, see `references/configurations.md`
3. **Does it use native UI on any platform?** (e.g. SwiftUI for iOS)
4. **What AGP version is in use?** If AGP 9.0+, the migration is mandatory for Android targets.
5. **Can they share their `build.gradle.kts` files?** Reading the actual files makes the migration more accurate.

---

### Step 2 — Create the `shared` module

Extract the KMP library portions from `composeApp`. Move all source sets (commonMain, androidMain, iosMain, desktopMain, wasmJsMain), shared dependencies, and resources. Do NOT move: `com.android.application` plugin, desktop `application {}` block, or app-specific signing/package config.

**`shared/build.gradle.kts` skeleton:**

```kotlin
plugins {
    alias(libs.plugins.kotlinMultiplatform)
    alias(libs.plugins.composeMultiplatform)
    alias(libs.plugins.composeCompiler)
    alias(libs.plugins.androidLibrary)   // library, not application
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
        }
    }
}

android {
    namespace = "com.example.myapp.shared"
    compileSdk = libs.versions.android.compileSdk.get().toInt()
}
```

---

### Step 3 — Create `androidApp` module

Standard Android application module that depends on `shared`.

```kotlin
plugins {
    alias(libs.plugins.androidApplication)
    alias(libs.plugins.kotlinAndroid)
    alias(libs.plugins.composeCompiler)
}

android {
    namespace = "com.example.myapp"
    defaultConfig {
        applicationId = "com.example.myapp"
        versionCode = 1
        versionName = "1.0"
    }
}

dependencies {
    implementation(projects.shared)
    implementation(libs.androidx.activity.compose)
}
```

Move `AndroidManifest.xml`, `MainActivity.kt`, and `res/` from `composeApp/src/androidMain/`. MainActivity just calls the shared `App()` composable.

---

### Step 4 — Create `desktopApp` module (if applicable)

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
    application { mainClass = "com.example.myapp.MainKt" }
}
```

Move `main()` from `composeApp/src/desktopMain/` into `desktopApp/src/main/kotlin/`.

---

### Step 5 — Create `webApp` module (if applicable)

```kotlin
plugins {
    alias(libs.plugins.kotlinMultiplatform)
    alias(libs.plugins.composeMultiplatform)
    alias(libs.plugins.composeCompiler)
}

kotlin {
    wasmJs {
        moduleName = "webApp"
        browser { commonWebpackConfig { outputFileName = "webApp.js" } }
        binaries.executable()
    }
    sourceSets {
        commonMain.dependencies { implementation(projects.shared) }
    }
}
```

---

### Step 6 — Update `settings.gradle.kts`

```kotlin
include(":shared")
include(":androidApp")
include(":desktopApp")   // if applicable
include(":webApp")       // if applicable
```

Remove the old `:composeApp` include.

---

### Step 7 — iOS: no structural change needed

`iosApp/` stays as-is. If you renamed the module, update the Xcode build phase:
```
./gradlew :shared:assembleXCFramework
```

---

### Step 8 — Clean up

- Delete the old `composeApp/` directory after verifying the build
- Update CI scripts referencing `:composeApp` tasks
- **Update IDE run configurations.** IntelliJ / Android Studio store run configurations under `.idea/runConfigurations/*.xml` and `.idea/workspace.xml`, and they still reference the old `:composeApp` Gradle tasks (`composeApp:installDebug`, `composeApp:run`, `composeApp:jsBrowserDevelopmentRun`, etc.). For each one:
  - Android app → change the Gradle task / module to `:androidApp` (typically `:androidApp:installDebug`) and update the module setting in **Run → Edit Configurations → Android App → Module**.
  - Desktop → change `composeApp:run` to `desktopApp:run`.
  - Web (JS / wasm) → change `composeApp:jsBrowserDevelopmentRun` / `wasmJsBrowserDevelopmentRun` to `:webApp:jsBrowserDevelopmentRun` / `:webApp:wasmJsBrowserDevelopmentRun`.
  - The pragmatic shortcut: delete the stale `.idea/runConfigurations/*.xml` entries (and the `<RunManager>` block in `.idea/workspace.xml` if needed) and let the IDE re-import the project — it will regenerate fresh configurations for the new modules. Commit the regenerated files so teammates don't hit the same issue.

---

## Common Issues

**"Cannot apply `com.android.application` to a multiplatform module"** → AGP 9.0 error. Move the app plugin to `androidApp`.

**Duplicate resources / `composeResources` not found** → Keep `compose.components.resources` only in `shared`; app modules depend on `projects.shared`.

**`expect`/`actual` not resolved** → Source set names in `shared` must match what they were in `composeApp`.

**iOS build fails after rename** → Check "Compile Kotlin Framework" build phase in Xcode for the old module name.

**Run configuration still launches `:composeApp` (or fails with "Task ':composeApp:…' not found")** → IDE run configurations under `.idea/runConfigurations/` and `.idea/workspace.xml` were not updated when the modules were renamed. Either edit each config's Gradle task / module fields to point at `:androidApp`, `:desktopApp`, or `:webApp`, or delete the stale XML files and let the IDE recreate them on next sync.

---

## References

- `references/configurations.md` — native UI and server-module variants
- Official migration guide: https://kotlinlang.org/docs/multiplatform/multiplatform-project-recommended-structure.html
- AGP 9.0 changes: https://blog.jetbrains.com/kotlin/2026/01/update-your-projects-for-agp9/
- KMP wizard: https://kmp.new
