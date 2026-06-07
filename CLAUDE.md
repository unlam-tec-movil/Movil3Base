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

Single-module Android app using **Hexagonal (Ports & Adapters) Architecture** with Jetpack Compose.

```
ar.edu.unlam.mobile.scaffolding/
│
├── domain/                     # Core business logic — NO Android, NO frameworks
│   ├── model/                  # Pure data models (Kotlin data classes only)
│   └── service/                # Pure domain services (stateless logic over models)
│
├── application/                # Orchestration & port definitions — NO Android, NO frameworks
│   ├── port/
│   │   ├── in/                 # Input ports: interfaces that ui calls (use cases)
│   │   └── out/                # Output ports: interfaces that data implements (repositories, remote sources)
│   └── service/                # Interactors: implement input ports, call output ports
│
├── ui/                         # Inbound adapter — Jetpack Compose + Android UI
│   ├── screens/
│   │   └── <feature>/
│   │       ├── <Feature>Screen.kt      # Composable layout
│   │       └── <Feature>ViewModel.kt  # Calls ONLY application.port.in interfaces
│   ├── components/             # Reusable stateless composables
│   ├── navigation/             # NavHost and route definitions
│   └── theme/                  # Material Design 3 (Color, Type, Theme)
│
└── data/                       # Outbound adapter — Room, Retrofit, sensors, camera
    ├── datasources/            # Room DAOs/Entities and Retrofit services
    ├── repositories/           # Implementations of application.port.out
    ├── mappers/                # Entity/DTO ↔ domain model converters
    └── di/                     # Hilt modules wiring ports to implementations
```

### Dependency Rules (strictly enforced)

| Layer         | May import              | Must NOT import          |
|---------------|-------------------------|--------------------------|
| `domain`      | nothing (pure Kotlin)   | application, ui, data    |
| `application` | domain only             | ui, data                 |
| `ui`          | application, domain     | data                     |
| `data`        | application, domain     | ui                       |

### Key Concepts

- **Input Port** (`application/port/in/`): interface defining a use case (e.g., `LoginUseCase`). UI calls these.
- **Output Port** (`application/port/out/`): interface defining a data contract (e.g., `UserRepository`, `LocationSource`). Data implements these.
- **Interactor** (`application/service/`): implements an input port, orchestrates domain logic and output ports.
- **Domain Service** (`domain/service/`): pure stateless logic over domain models. No I/O, no Android.

### Navigation

Single-activity (`MainActivity`) with a `NavHost` inside `MainScreen()`. Routes are string constants; arguments are passed as path segments (e.g., `user/{id}`).

### Dependency Injection

Dagger Hilt throughout. `ScaffoldingApplication` is `@HiltAndroidApp`, `MainActivity` is `@AndroidEntryPoint`, ViewModels use `@HiltViewModel` + `hiltViewModel()` at call sites.

## CI / PR Requirements

Two GitHub Actions workflows run on pull requests:

- **android-lint.yml** — runs `ktlint` and `./gradlew lint`; triggers only when code or Gradle files change.
- **test-coverage.yml** — runs Kover and enforces **60% line coverage** on both overall and changed files; posts a coverage comment on the PR.

PRs must pass both workflows before merging.

