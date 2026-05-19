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

### Decision rule

> If **all** platforms use this code (including native UI platforms): `sharedLogic`.
> If only **Compose Multiplatform** platforms need it: `sharedUI`.

---

## With Server Module

Adds a `server` module and groups client-side modules under an `app/` subfolder:

```
project/
├── core/                 ← Shared between server AND clients
├── server/               ← Ktor server
└── app/
    ├── shared/
        ├── androidApp/
            ├── desktopApp/
                ├── webApp/
                    └── iosApp/
                    ```

                    `settings.gradle.kts` for server projects:

                    ```kotlin
                    include(":core")
                    include(":server")
                    include(":app:shared")
                    include(":app:androidApp")
                    include(":app:desktopApp")
                    include(":app:webApp")
                    ```

                    ---

                    ## Summary

                    | Situation | Structure |
                    |---|---|
                    | Compose UI on all platforms | `shared` + app modules |
                    | Native UI on ≥1 platform | `sharedLogic` + `sharedUI` + app modules |
                    | Includes a server | `core/` at root, clients under `app/` |
                    | Combinations | Compose these patterns |
