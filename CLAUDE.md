# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
# Build
./gradlew assembleDebug
./gradlew assembleRelease

# Tests
./gradlew test                          # Unit tests
./gradlew connectedAndroidTest          # Instrumented tests (requires device/emulator)

# Code quality
./gradlew lint                          # Android lint
./gradlew :app:ktlint                   # ktlint style check
./gradlew :app:ktlintFormat             # Auto-fix ktlint issues

# Coverage
./gradlew :app:koverXmlReportRelease    # Generate coverage report (min 60% required)
```

## Architecture

Single-module Android app using **Hexagonal / Clean Architecture** with Jetpack Compose.

### Layers

**UI Layer** (`ui/`)
- `screens/` — Screen-level composables paired 1:1 with a ViewModel. Each screen owns its state via `StateFlow` and exposes a sealed `UiState` (Loading / Success / Error).
- `components/` — Stateless, reusable composables that receive data and lambdas.
- `theme/` — Material Design 3 theming (Color, Type, Theme).

**Data Layer** (`data/`)
- `datasources/network/` — Retrofit-based remote sources.
- `datasources/local/` — Room-based local sources.
- `repositories/` — Abstractions that combine data sources; consumed by ViewModels.
- `di/` — Hilt modules wiring datasources and repositories.

### Navigation

Single-activity (`MainActivity`) with a `NavHost` inside `MainScreen()`. Routes are string constants; arguments are passed as path segments (e.g., `user/{id}`).

### Dependency Injection

Dagger Hilt throughout. `ScaffoldingApplication` is `@HiltAndroidApp`, `MainActivity` is `@AndroidEntryPoint`, ViewModels use `@HiltViewModel` + `hiltViewModel()` at call sites.

## CI / PR Requirements

Two GitHub Actions workflows run on pull requests:

- **android-lint.yml** — runs `ktlint` and `./gradlew lint`; triggers only when code or Gradle files change.
- **test-coverage.yml** — runs Kover and enforces **60% line coverage** on both overall and changed files; posts a coverage comment on the PR.

PRs must pass both workflows before merging.
