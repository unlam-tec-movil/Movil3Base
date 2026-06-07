# Application Layer

**Regla: puede importar `domain`. No puede importar `presentation` ni `infrastructure`.**

Esta capa define **qué puede hacer el sistema** (puertos) y **cómo se orquesta** (interactors).
Es el puente entre el mundo exterior y el núcleo de negocio.

## Estructura

```
application/
├── port/
│   ├── in/         # Puertos de entrada (casos de uso)
│   └── out/        # Puertos de salida (repositorios, fuentes de datos)
└── service/        # Interactors (implementan port/in, usan port/out)
```

## `port/in/` — Puertos de Entrada

Interfaces que exponen las acciones que el sistema puede realizar. La capa `presentation` las
inyecta en los ViewModels. **Nunca se inyectan clases concretas de infrastructure.**

```kotlin
interface LoginUseCase {
    suspend operator fun invoke(email: String, password: String): String
}
```

Un puerto de entrada define *qué* se hace, no *cómo*. Sus parámetros y retornos deben ser tipos
de dominio o primitivos; nunca tipos de Android o de infrastructure.

## `port/out/` — Puertos de Salida

Interfaces que define la application layer y que la capa `infrastructure` implementa. Son
contratos de I/O vistos desde el negocio:

```kotlin
interface UserRepository {
    fun getUser(): Flow<User?>
    suspend fun saveUser(user: User)
    suspend fun clearUser()
}

interface LocationSource {
    fun getLocationUpdates(): Flow<Location>
}
```

> **Nota sobre Android types en port/out**: tipos como `android.location.Location` pueden aparecer
> en puertos de salida si representan un concepto de dominio sin alternativa pura. Documentar la
> excepción; evitarla cuando sea posible usando un modelo de dominio propio.

## `service/` — Interactors

Implementan los puertos de entrada. Orquestan domain services y puertos de salida. No tienen
estado propio; son stateless.

```kotlin
class LoginInteractor @Inject constructor(
    private val userRepository: UserRepository,   // port/out
    private val sessionSource: SessionSource,      // port/out
    private val authService: AuthService,          // domain service o port/out externo
) : LoginUseCase {
    override suspend fun invoke(email: String, password: String): String {
        val token = authService.authenticate(email, password)
        userRepository.saveUser(User(id = token.uid, ...))
        return token.value
    }
}
```

## Qué NO vive aquí

- Clases de Room, Retrofit, Firebase, CameraX, SharedPreferences → van en `infrastructure/`
- Composables, ViewModels, Activities → van en `presentation/`
- Modelos de dominio → van en `domain/model/`
- Lógica pura de negocio sin I/O → va en `domain/service/`

## Relación con otras capas

```
presentation  ──► application/port/in  (llama use cases)
application   ──► domain               (usa modelos y servicios)
infrastructure ──► application/port/out (implementa repositorios)
```
