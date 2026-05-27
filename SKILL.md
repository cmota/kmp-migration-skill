---
name: kmp-migration
description: >
  Use this skill whenever a user wants to migrate a Kotlin Multiplatform (KMP) or Compose
  Multiplatform project from the old single-module structure (with a `composeApp` module) to
  the new default structure (with a `shared` library module and separate `androidApp`,
  `desktopApp`, `webApp` application modules). Also trigger for questions about the new KMP
  project structure, what changed in the KMP wizard, how to split `composeApp` into separate
  modules, stale Android Studio or IntelliJ run configurations that still show `composeApp`,
  or how to comply with Android Gradle Plugin (AGP) 9.0's requirement to separate app and
  library code. Use this even if the user phrases it casually, e.g. "how do I update my KMP
  project structure", "split my composeApp module", "composeApp still appears in Edit
  Configurations", or "AGP 9.0 breaking my KMP build".
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
5. **Can they share their `settings.gradle.kts` and `build.gradle.kts` files?** Reading the actual files makes the migration more accurate.
6. **Does the IDE still show `composeApp` after migration?** If yes, inspect `.idea` metadata as part of cleanup.

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

If `composeApp` still appears in Android Studio / IntelliJ, verify this file first. The IDE
will keep importing `composeApp` as long as `settings.gradle.kts` still contains:

```kotlin
include(":composeApp")
```

Replace it with only the modules that actually exist. Do not add `desktopApp` or `webApp`
unless those modules were created.

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
- Search the project for stale module references before declaring the migration done:

```bash
rg -n "composeApp" . --hidden \
  --glob 'settings.gradle.kts' \
  --glob '**/*.gradle.kts' \
  --glob '.idea/**' \
  --glob '.github/**' \
  --glob 'gradle/**'
```

- **Update IDE metadata and run configurations.** IntelliJ / Android Studio may keep showing
  `composeApp` even after the source migration if any of these files still reference it:
  - `.idea/gradle.xml` module entries such as `$PROJECT_DIR$/composeApp`
  - `.idea/runConfigurations/*.xml` and `.idea/workspace.xml` run manager entries
  - `.idea/deploymentTargetSelector.xml` entries such as `runConfigName="composeApp"`
  - `.idea/artifacts/composeApp_*.xml` generated artifact definitions
- For each stale run task:
  - Android app: change `:composeApp:installDebug` / `composeApp:installDebug` to `:androidApp:installDebug`, and update **Run → Edit Configurations → Android App → Module** to `androidApp`.
  - Desktop: change `:composeApp:run` / `composeApp:run` to `:desktopApp:run`.
  - Web: change `:composeApp:jsBrowserDevelopmentRun`, `:composeApp:wasmJsBrowserDevelopmentRun`, or unqualified equivalents to the matching `:webApp:*BrowserDevelopmentRun` task.
- If the IDE files are not intentionally versioned, the pragmatic shortcut is to close the IDE,
  delete stale `.idea/runConfigurations/*.xml`, `.idea/artifacts/composeApp_*.xml`, and the
  `composeApp` entries in `.idea/gradle.xml`, `.idea/deploymentTargetSelector.xml`, and
  `.idea/workspace.xml`, then re-open and sync Gradle.
- Commit updated shared IDE files if the repository tracks them, so teammates do not keep
  importing or launching `composeApp`.

---

### Step 9 — Sync and verify before finishing

Do not end the migration immediately after editing files. Finish with a project synchronization
and validation pass:

1. Run `./gradlew projects` and confirm the output lists the new modules and does not list
   `:composeApp`.
2. Run the relevant compile/build tasks for the targets that exist, for example:

```bash
./gradlew :shared:compileKotlinMetadata
./gradlew :androidApp:assembleDebug
./gradlew :desktopApp:compileKotlin
./gradlew :webApp:compileKotlinJs
./gradlew :webApp:compileKotlinWasmJs
./gradlew :shared:compileKotlinIosSimulatorArm64
```

3. Search for stale module references one last time:

```bash
rg -n "composeApp|ComposeApp" . --hidden --glob '!**/build/**' --glob '!**/.gradle/**' --glob '!**/.kotlin/**'
```

4. In the final response, explicitly tell the user to run **File → Sync Project with Gradle
   Files** in Android Studio / IntelliJ, or close and reopen the IDE if stale `composeApp`
   configurations still appear. There is no reliable repository-only edit that forces an already
   open IDE window to complete a Gradle sync.

---

## Common Issues

**"Cannot apply `com.android.application` to a multiplatform module"** → AGP 9.0 error. Move the app plugin to `androidApp`.

**Duplicate resources / `composeResources` not found** → Keep `compose.components.resources` only in `shared`; app modules depend on `projects.shared`.

**`expect`/`actual` not resolved** → Source set names in `shared` must match what they were in `composeApp`.

**iOS build fails after rename** → Check "Compile Kotlin Framework" build phase in Xcode for the old module name.

**`composeApp` still appears in Edit Configurations or the Gradle tool window** → First check `settings.gradle.kts`; if it still has `include(":composeApp")`, replace it with `include(":shared")`, `include(":androidApp")`, and any created `desktopApp` / `webApp` modules. Then search `.idea` for `composeApp`. Remove or update stale entries in `.idea/gradle.xml`, `.idea/runConfigurations/*.xml`, `.idea/workspace.xml`, `.idea/deploymentTargetSelector.xml`, and `.idea/artifacts/composeApp_*.xml`, then re-sync Gradle.

**Run configuration still launches `:composeApp` (or fails with "Task ':composeApp:…' not found")** → IDE run configurations were not updated when the modules were renamed. Either edit each config's Gradle task / module fields to point at `:androidApp`, `:desktopApp`, or `:webApp`, or delete the stale XML files and let the IDE recreate them on next sync.

**Edits look correct but the IDE still shows old modules** → Run **File → Sync Project with
Gradle Files**. If Android Studio / IntelliJ was open during the migration and still shows
stale entries after sync, close and reopen the project so it reloads `settings.gradle.kts` and
the updated `.idea` metadata.

---

## References

- `references/configurations.md` — native UI and server-module variants
- Official migration guide: https://kotlinlang.org/docs/multiplatform/multiplatform-project-recommended-structure.html
- AGP 9.0 changes: https://blog.jetbrains.com/kotlin/2026/01/update-your-projects-for-agp9/
- KMP wizard: https://kmp.new
