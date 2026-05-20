# kmp-migration-skill

An [agent skill](https://geminicli.com/docs/cli/skills/) for migrating Kotlin Multiplatform (KMP) projects from the old `composeApp` single-module structure to the new default structure introduced in May 2026. Compatible with Claude Code, GitHub Copilot, OpenAI Codex, and Gemini CLI.

## What this skill does

Guides the agent through helping you migrate a KMP/Compose Multiplatform project from this:

```
project/
└── composeApp/   ← single module acting as both library and app entry point
```

To this:

```
project/
├── shared/       ← KMP library with all shared code
├── androidApp/   ← Android application module
├── desktopApp/   ← Desktop application module
├── webApp/       ← Web (WASM) application module
└── iosApp/       ← Xcode project (unchanged)
```

## Why migrate?

The new structure is the official default as of the [KMP wizard](https://kmp.new) update in May 2026. The key driver is **Android Gradle Plugin 9.0**, which requires the Android app entry point to be in a separate module from shared code.

See the [JetBrains announcement](https://blog.jetbrains.com/kotlin/2026/05/new-kmp-default-structure/) for full details.

## Skill contents

- `skills/kmp-migration/SKILL.md` — main migration guide with step-by-step instructions for each module
- `skills/kmp-migration/references/configurations.md` — variant structures for native UI (e.g. SwiftUI on iOS) and server-included projects

## Installation

> `SKILL.md` is an open standard — the skill works across all compatible agents. The only difference between tools is the installation path.

### Claude Code

```bash
/plugin marketplace add cmota/kmp-migration-skill
/plugin install kmp-migration@kmp-migration-skill
```

### GitHub Copilot

```bash
gh skills install cmota/kmp-migration-skill kmp-migration
```

Or manually — copy `skills/kmp-migration/` into `.github/skills/` in your repo (project scope) or `~/.copilot/skills/` (user scope).

### OpenAI Codex

```bash
$skill-installer install https://github.com/cmota/kmp-migration-skill/tree/main/skills/kmp-migration
```

### Gemini CLI

```bash
gemini skills install https://github.com/cmota/kmp-migration-skill.git --path skills/kmp-migration
```

## Covered scenarios

- Standard Compose Multiplatform (Android + iOS + Desktop + Web)
- Native UI on some platforms (splits into `sharedLogic` + `sharedUI`)
- Projects with a server module (adds `core/` + nests clients under `app/`)
- AGP 9.0 compliance

## References

- [Official KMP migration guide](https://kotlinlang.org/docs/multiplatform/multiplatform-project-recommended-structure.html)
- [AGP 9.0 update guide](https://blog.jetbrains.com/kotlin/2026/01/update-your-projects-for-agp9/)
- [KMP wizard](https://kmp.new)
