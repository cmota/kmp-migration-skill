# KMP Migration: Variant Configurations

## Native UI on Some Platforms (e.g. SwiftUI for iOS)

When one or more platforms use native UI instead of Compose Multiplatform, the new structure
uses **two** shared modules:

```
project/
├── sharedLogic/          ← Pure KMP library, no Compose, all platforms consume it
├── sharedUI/             ← KMP library with Compose, consumed only by Compose platforms
├── androidApp/
├── desktopApp/
├── webApp/
└── iosApp/               ← Uses SwiftUI; depends on sharedLogic, NOT sharedUI
```

### `sharedLogic/build.gradle.kts`

```kotlin
plugins {
    alias(libs.plugins.kotlinMultiplatform)
    alias(libs.plugins.androidLibrary)
}

kotlin {
    androidTarget()
    iosX64(); iosArm64(); iosSimulatorArm64()
    jvm("desktop")
    wasmJs { browser() }

    sourceSets {
        commonMain.dependencies {
            // Data models, repositories, use cases — NO Compose imports
            implementation(libs.kotlinx.coroutines.core)
            implementation(libs.kotlinx.serialization.json)
        }
    }
}
```

### `sharedUI/build.gradle.kts`

```kotlin
plugins {
    alias(libs.plugins.kotlinMultiplatform)
    alias(libs.plugins.composeMultiplatform)
    alias(libs.plugins.composeCompiler)
    alias(libs.plugins.androidLibrary)
}

kotlin {
    androidTarget()
    jvm("desktop")
    wasmJs { browser() }
    // Note: NO iosX64/iosArm64 here — iOS uses SwiftUI, not this module

    sourceSets {
        commonMain.dependencies {
            implementation(projects.sharedLogic)
            implementation(compose.runtime)
            implementation(compose.material3)
            // etc.
        }
    }
}
```

### Decision rule for developers

> If **all** platforms will use this code (including those with native UI): put it in `sharedLogic`.
> If only **Compose Multiplatform** platforms need it: put it in `sharedUI`.

---

## With Server Module

When the project also targets server-side Kotlin (e.g. Ktor), the structure adds a top-level
`server` module and groups client-side modules under an `app/` subfolder:

```
project/
├── core/                 ← Shared between server AND clients (models, validation, API contracts)
├── server/               ← Ktor (or other) server application
└── app/
    ├── shared/           ← Client-side KMP library (Compose UI, client logic)
    ├── androidApp/
    ├── desktopApp/
    ├── webApp/
    └── iosApp/           ← Xcode project
```

### `core/` module

The `core` module is a pure Kotlin Multiplatform library with no platform dependencies beyond
what's needed for shared data contracts:

```kotlin
// core/build.gradle.kts
plugins {
    alias(libs.plugins.kotlinMultiplatform)
}

kotlin {
    jvm()           // server consumes this
    androidTarget() // clients consume this
    iosX64(); iosArm64(); iosSimulatorArm64()
    wasmJs { browser() }

    sourceSets {
        commonMain.dependencies {
            implementation(libs.kotlinx.serialization.json)
            // Shared models, API response types, validation logic
        }
    }
}
```

### `server/build.gradle.kts`

```kotlin
plugins {
    alias(libs.plugins.kotlinJvm)
    alias(libs.plugins.ktor)
}

dependencies {
    implementation(projects.core)
    implementation(libs.ktor.server.core)
    implementation(libs.ktor.server.netty)
    // ...
}
```

### `settings.gradle.kts` for server projects

```kotlin
include(":core")
include(":server")
include(":app:shared")
include(":app:androidApp")
include(":app:desktopApp")
include(":app:webApp")
```

Note: `:app:shared` and `:app:androidApp` etc. use the nested path syntax. The `iosApp`
Xcode project lives at `app/iosApp/` but is not a Gradle module.

---

## Summary: Which Structure Do I Need?

| Situation | Structure |
|---|---|
| Compose UI on all platforms | `shared` + `androidApp` + `desktopApp` + `webApp` |
| Native UI on ≥1 platform | `sharedLogic` + `sharedUI` + per-platform app modules |
| Includes a server | Add `core/` at root, nest client modules under `app/` |
| Combinations | Compose these patterns (e.g. server + native iOS UI: `core`, `server`, `app/sharedLogic`, `app/sharedUI`, ...) |
