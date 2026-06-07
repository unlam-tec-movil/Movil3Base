# Mobile Scaffolding

## Índice

- [Configuración](#configuración)
- [Arquitectura Hexagonal](#arquitectura-hexagonal)
  - [Domain](#domain)
  - [Application](#application)
  - [UI](#ui)
  - [Infrastructure](#infrastructure)
  - [Reglas de dependencia](#reglas-de-dependencia)
- [Navegación](#navegación)
- [Inyección de Dependencias](#inyección-de-dependencias)

## Consideraciones previas

Para este documento usaremos como ejemplo una aplicación de rehabilitación física. Los modelos de
negocio centrales son: `Session`, `Exercise`, `User` y `Clinic`.

## Configuración

El proyecto ejecuta dos workflows de GitHub en cada PR:

1. Análisis estático con `ktlint` y `./gradlew lint`.
2. Tests con cobertura mínima del 60% (Kover).

Para configurar el proyecto en Android Studio, instalar el plugin
de [ktlint](https://plugins.jetbrains.com/plugin/15057-ktlint).

---

## Arquitectura Hexagonal

El proyecto implementa la **Arquitectura Hexagonal (Ports & Adapters)** en un único módulo Android.
El objetivo es aislar la lógica de negocio del framework, haciendo el core testeable sin emulador.

```
ar.edu.unlam.mobile.scaffolding/
│
├── domain/                     # Núcleo de negocio — sin Android, sin frameworks
│   ├── model/                  # Modelos de dominio puros (data classes Kotlin)
│   └── service/                # Servicios de dominio: lógica pura sobre modelos
│
├── application/                # Orquestación y definición de puertos
│   ├── port/
│   │   ├── in/                 # Puertos de entrada: interfaces que llama UI
│   │   └── out/                # Puertos de salida: interfaces que implementa Infrastructure
│   └── service/                # Interactors: implementan port/in, usan port/out
│
├── ui/                         # Adaptador de entrada — Jetpack Compose, ViewModels
│   ├── screens/
│   │   └── <feature>/
│   │       ├── <Feature>Screen.kt
│   │       └── <Feature>ViewModel.kt
│   ├── components/             # Composables reutilizables sin estado propio
│   ├── navigation/             # Definición de rutas y NavHost
│   └── theme/                  # Material Design 3 (Color, Type, Theme)
│
└── infrastructure/             # Adaptador de salida — Room, Retrofit, sensores, cámara
    ├── db/                     # Room: implementa application.port.out
    ├── network/                # Retrofit: implementa application.port.out
    ├── di/                     # Módulos Hilt
    └── <adapter>/              # Adaptadores de dispositivo (cámara, sensores, ubicación)
```

---

### Domain

La capa más interna. Contiene el conocimiento de negocio puro, sin dependencias externas.

**`model/`** — Modelos de dominio representados como `data class` de Kotlin. No tienen lógica de
persistencia ni de presentación.

**`service/`** — Servicios de dominio: funciones puras que operan sobre los modelos. No tienen
estado, no hacen I/O, no importan Android ni frameworks.

Ejemplo de servicio de dominio:

```kotlin
class AngleComparisonService @Inject constructor() {
    fun compare(measured: Float, target: Float): Feedback { ... }
}
```

---

### Application

Capa de orquestación. Define **qué** puede hacer el sistema (puertos) y **cómo** se coordina
(interactors). No tiene dependencias de Android ni de infrastructure.

**`port/in/`** — Interfaces de casos de uso que la capa de UI invoca. Cada interfaz
representa una acción del usuario o del sistema:

```kotlin
interface LoginUseCase {
    suspend operator fun invoke(email: String, password: String): String
}
```

**`port/out/`** — Interfaces de repositorios y fuentes de datos que infrastructure implementa:

```kotlin
interface UserRepository {
    suspend fun saveUser(user: User)
    fun getUser(): Flow<User?>
}
```

**`service/`** — Interactors: implementan los puertos de entrada, llaman a los puertos de salida y
orquestan servicios de dominio:

```kotlin
class LoginInteractor @Inject constructor(
    private val userRepository: UserRepository,   // port/out
    private val sessionSource: SessionSource,      // port/out
) : LoginUseCase {
    override suspend fun invoke(email: String, password: String): String { ... }
}
```

---

### UI

Adaptador de entrada. Contiene toda la UI de Android (Jetpack Compose, ViewModels, Navigation).

- Los **ViewModels** inyectan interfaces de `application/port/in/` — nunca clases de infrastructure.
- Las pantallas son composables **sin estado propio**; el estado viene del ViewModel via `StateFlow`.
- Los componentes reutilizables viven en `ui/components/`.

#### Paradigma Contenedor / Componente

Los composables se organizan en dos tipos:

- **Screen (contenedor)**: orquesta componentes, conecta con el ViewModel, maneja navegación.
- **Component**: recibe estado y lambdas, no conoce el ViewModel ni la navegación.

#### Navegación

Single-activity con `NavHost` en `MainScreen`. Rutas como constantes de string; argumentos como
segmentos de path:

```kotlin
NavHost(navController = controller, startDestination = "home") {
    composable("home") { HomeScreen() }
    composable(
        route = "rehab/{sessionId}",
        arguments = listOf(navArgument("sessionId") { type = NavType.StringType }),
    ) { backStackEntry ->
        val id = backStackEntry.arguments?.getString("sessionId") ?: ""
        RehabSessionScreen(sessionId = id)
    }
}
```

---

### Infrastructure

Adaptador de salida. Implementa los puertos definidos en `application/port/out/` usando
tecnologías concretas (Room, Retrofit, Firebase, sensores Android).

**`db/`** — Implementaciones Room (DAOs, Entities, Database, Mappers).

**`network/`** — Clientes Retrofit (API interfaces, DTOs, Mappers).

**`di/`** — Módulos Hilt que ligan interfaces (`port/out`) con sus implementaciones.

**`<adapter>/`** — Adaptadores de dispositivo: cámara (CameraX), sensores, ubicación.

---

### Reglas de dependencia

| Capa             | Puede importar          | NO puede importar               |
|------------------|-------------------------|---------------------------------|
| `domain`         | nada (Kotlin puro)      | application, ui, infrastructure |
| `application`    | domain únicamente       | ui, infrastructure              |
| `ui`             | application, domain     | infrastructure                  |
| `infrastructure` | application, domain     | ui                              |

Estas reglas se deben verificar en code review. Una violación típica es inyectar una clase
concreta de infrastructure (ej. `SessionPreferences`) directamente en un interactor de application.

---

## Inyección de Dependencias

Dagger Hilt en toda la aplicación:

- `ScaffoldingApplication` → `@HiltAndroidApp`
- `MainActivity` → `@AndroidEntryPoint`
- ViewModels → `@HiltViewModel` + `hiltViewModel()` en el composable
- Los módulos Hilt (en `infrastructure/di/`) ligan cada interfaz `port/out` con su implementación

---

## Referencias

[1]: https://martinfowler.com/bliki/DomainDrivenDesign.html
[2]: https://developer.android.com/training/dependency-injection/hilt-android
[3]: https://alistair.cockburn.us/hexagonal-architecture/

