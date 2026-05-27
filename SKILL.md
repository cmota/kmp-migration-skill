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
- iOS was already in a separate `iosApp` folder â the other platforms were inconsistently co-located
- Amper-based projects already used separate modules per app; Gradle projects were out of step

The new structure separates concerns clearly. For variant configurations (native UI, server), see `references/configurations.md`.

---

## Migration Workflow

### Step 1 â Understand the user's project

Before doing anything, ask or infer:

1. **Which targets does the project support?** (Android, iOS, desktop, web/WASM)
2. **Does it have a server module?** If yes, see `references/configurations.md`
3. **Does it use native UI on any platform?** (e.g. SwiftUI for iOS)
4. **What AGP version is in use?** If AGP 9.0+, the migration is mandatory for Android targets.
5. **Can they share their `settings.gradle.kts` and `build.gradle.kts` files?** Reading the actual files makes the migration more accurate.
6. **Does the IDE still show `composeApp` after migration?** If yes, inspect `.idea` metadata as part of cleanup.

---

### Step 2 â Create the `shared` module

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

### Step 3 â Create `androidApp` module

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

### Step 4 â Create `desktopApp` module (if applicable)

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

### Step 5 â Create `webApp` module (if applicable)

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

### Step 6 â Update `settings.gradle.kts`

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

### Step 7 â Update iOS references

`iosApp/` stays as-is structurally, but renaming the module concept from `composeApp` to
`shared` requires updating several iOS-side references. Both lowercase `composeApp` **and**
uppercase `ComposeApp` must be searched â the uppercase form is used as the framework
`baseName` and Swift import.

**7a â Update the Gradle framework `baseName`**

In `shared/build.gradle.kts` (or the root build file), find the XCFramework / framework
configuration and change the `baseName`:

```kotlin
// Before
kotlin {
    iosX64 { binaries.framework { baseName = "ComposeApp" } }
    // ...
}

// After
kotlin {
    iosX64 { binaries.framework { baseName = "Shared" } }
    // ...
}
```

**7b â Update the Xcode build phase**

In Xcode â Build Phases â "Compile Kotlin Framework", update the Gradle task:
```
./gradlew :shared:assembleXCFramework
```

**7c â Update Swift imports**

Search `iosApp/` for `import ComposeApp` and replace with `import Shared`:

```bash
rg -n "ComposeApp" iosApp/ --glob '*.swift'
```

Update every occurrence in `.swift` files â typically `iOSApp.swift` and `ContentView.swift`.

**7d â Update Podfile or SPM manifest (if applicable)**

If using CocoaPods, update the pod name in `iosApp/Podfile`:
```ruby
# Before
pod 'ComposeApp', :path => '../'

# After
pod 'Shared', :path => '../'
```

If using Swift Package Manager, update the target dependency name accordingly.

**7e â Verify with `xcodebuild`, not just Gradle**

A successful `./gradlew :shared:assembleXCFramework` does not guarantee the Xcode build
works. Always verify with an actual Xcode build:

```bash
xcodebuild -project iosApp/iosApp.xcodeproj \
  -scheme iosApp \
  -destination 'platform=iOS Simulator,name=iPhone 15' \
  build
```

Or simply build from Xcode IDE to catch any remaining `ComposeApp` references.

---

### Step 8 â Clean up

- Delete the old `composeApp/` directory after verifying the build
- Update CI scripts referencing `:composeApp` tasks
- Search the project for stale module references before declaring the migration done:

```bash
rg -n "composeApp|ComposeApp" . --hidden \
  --glob 'settings.gradle.kts' \
  --glob '**/*.gradle.kts' \
  --glob '.idea/**' \
  --glob '.github/**' \
  --glob 'gradle/**' \
  --glob 'iosApp/**'
```

- **Update IDE metadata and run configurations.** IntelliJ / Android Studio may keep showing
  `composeApp` even after the source migration if any of these files still reference it:
  - `.idea/gradle.xml` module entries such as `$PROJECT_DIR$/composeApp`
  - `.idea/runConfigurations/*.xml` and `.idea/workspace.xml` run manager entries
  - `.idea/deploymentTargetSelector.xml` entries such as `runConfigName="composeApp"`
  - `.idea/artifacts/composeApp_*.xml` generated artifact definitions
- **Exact run configuration field mappings** â update each stale config as follows:

  | Field | Old value | New value |
  |---|---|---|
  | Android run config name | `Android App.composeApp` | `Android App.androidApp` |
  | Android module field | `Breeze.composeApp` | `Breeze.androidApp` |
  | Desktop config (hot reload) | `composeApp [jvm, hot]` | `desktopApp [jvm, hot]` |
  | Desktop config (standard) | `composeApp [jvm]` | `desktopApp [jvm]` |
  | Web config (JS) | `composeApp [js]` | `webApp [js]` |
  | Web config (WASM) | `composeApp [wasmJs]` | `webApp [wasmJs]` |
  | Gradle project path (desktop) | `$PROJECT_DIR$/composeApp` | `$PROJECT_DIR$/desktopApp` |
  | Gradle project path (web) | `$PROJECT_DIR$/composeApp` | `$PROJECT_DIR$/webApp` |
  | Desktop hot-reload task | `hotRunJvm` | `hotRun` |
  | Desktop standard task | `jvmRun` | `run` |

- If the IDE files are not intentionally versioned, the pragmatic shortcut is to close the IDE,
  delete stale `.idea/runConfigurations/*.xml`, `.idea/artifacts/composeApp_*.xml`, and the
  `composeApp` entries in `.idea/gradle.xml`, `.idea/deploymentTargetSelector.xml`, and
  `.idea/workspace.xml`, then re-open and sync Gradle.
- Commit updated shared IDE files if the repository tracks them, so teammates do not keep
  importing or launching `composeApp`.

---

### Step 9 â Sync and verify before finishing

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

3. For iOS, verify with `xcodebuild` (see Step 7e) in addition to the Gradle tasks above.
4. Search for stale module references one last time:

```bash
rg -n "composeApp|ComposeApp" . --hidden --glob '!**/build/**' --glob '!**/.gradle/**' --glob '!**/.kotlin/**'
```

5. In the final response, explicitly tell the user to run **File â Sync Project with Gradle
   Files** in Android Studio / IntelliJ, or close and reopen the IDE if stale `composeApp`
   configurations still appear. There is no reliable repository-only edit that forces an already
   open IDE window to complete a Gradle sync.

---

## Common Issues

**"Cannot apply `com.android.application` to a multiplatform module"** â AGP 9.0 error. Move the app plugin to `androidApp`.

**Duplicate resources / `composeResources` not found** â Keep `compose.components.resources` only in `shared`; app modules depend on `projects.shared`.

**`expect`/`actual` not resolved** â Source set names in `shared` must match what they were in `composeApp`.

**iOS build fails after rename** â Search for both `composeApp` and `ComposeApp` in `iosApp/`. The uppercase form is the framework `baseName` and Swift import â update `baseName = "Shared"` in the Gradle build file, replace `import ComposeApp` with `import Shared` in all Swift files, update any Podfile or SPM references, and verify with `xcodebuild` rather than Gradle alone.

**`composeApp` still appears in Edit Configurations or the Gradle tool window** â First check `settings.gradle.kts`; if it still has `include(":composeApp")`, replace it with `include(":shared")`, `include(":androidApp")`, and any created `desktopApp` / `webApp` modules. Then search `.idea` for `composeApp`. Remove or update stale entries in `.idea/gradle.xml`, `.idea/runConfigurations/*.xml`, `.idea/workspace.xml`, `.idea/deploymentTargetSelector.xml`, and `.idea/artifacts/composeApp_*.xml`, then re-sync Gradle.

**Run configuration still launches `:composeApp` (or fails with "Task ':composeApp:â¦' not found")** â IDE run configurations were not updated. Use the field mapping table in Step 8 to update each config's name, module, Gradle project path, and task name. Or delete the stale XML files and let the IDE recreate them on next sync.

**Edits look correct but the IDE still shows old modules** â Run **File â Sync Project with
Gradle Files**. If Android Studio / IntelliJ was open during the migration and still shows
stale entries after sync, close and reopen the project so it reloads `settings.gradle.kts` and
the updated `.idea` metadata.

---

## References

- `references/configurations.md` â native UI and server-module variants
- Official migration guide: https://kotlinlang.org/docs/multiplatform/multiplatform-project-recommended-structure.html
- AGP 9.0 changes: https://blog.jetbrains.com/kotlin/2026/01/update-your-projects-for-agp9/
- KMP wizard: https://kmp.new
