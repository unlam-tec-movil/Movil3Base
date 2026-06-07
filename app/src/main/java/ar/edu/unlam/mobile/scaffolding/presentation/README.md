# Presentation Layer

**Regla: puede importar `application` y `domain`. No puede importar `infrastructure`.**

Esta capa es el adaptador de entrada. Contiene toda la UI de Android: composables, ViewModels
y configuración de navegación. Es la única capa que conoce Jetpack Compose, Activities y
el ciclo de vida de Android.

## Estructura

```
presentation/
├── <feature>/
│   ├── <Feature>Screen.kt      # Composable de pantalla (layout)
│   └── <Feature>ViewModel.kt   # Estado y lógica de UI
├── components/                 # Composables reutilizables sin estado propio
├── navigation/                 # Definición de rutas y NavHost
└── theme/                      # Material Design 3 (Color, Type, Theme)
```

## ViewModels

Los ViewModels son los únicos responsables de mantener el estado de la UI. Inyectan exclusivamente
interfaces de `application/port/in/` — **nunca clases concretas de infrastructure**.

```kotlin
@HiltViewModel
class LoginViewModel @Inject constructor(
    private val loginUseCase: LoginUseCase,   // application/port/in — interfaz, no la clase concreta
) : ViewModel() {

    private val _uiState = MutableStateFlow<UiState>(UiState.Idle)
    val uiState: StateFlow<UiState> = _uiState.asStateFlow()

    fun login(email: String, password: String) {
        viewModelScope.launch {
            _uiState.value = UiState.Loading
            runCatching { loginUseCase(email, password) }
                .onSuccess { _uiState.value = UiState.Success }
                .onFailure { _uiState.value = UiState.Error(it.message) }
        }
    }
}
```

## Screens (Contenedores)

Composables de nivel de pantalla. Orquestan componentes, leen el estado del ViewModel y manejan
eventos de navegación. Se anotan con el ViewModel como parámetro por defecto vía `hiltViewModel()`:

```kotlin
@Composable
fun LoginScreen(
    modifier: Modifier = Modifier,
    viewModel: LoginViewModel = hiltViewModel(),
    onLoginSuccess: () -> Unit,
) {
    val state by viewModel.uiState.collectAsStateWithLifecycle()
    LoginContent(state = state, onLogin = viewModel::login)
}
```

## Components (Componentes)

Composables reutilizables, **stateless**. Reciben datos y lambdas; no conocen ViewModels ni
navegación. Pueden reutilizarse en distintas pantallas.

## Paradigma Contenedor / Componente

| Tipo       | Conoce el ViewModel | Maneja navegación | Tiene estado propio |
|------------|---------------------|-------------------|---------------------|
| Screen     | Sí                  | Sí                | No (lo delega al VM)|
| Component  | No                  | No                | No                  |

## UiState

Cada pantalla expone un `sealed class UiState` con al menos tres estados:

```kotlin
sealed class UiState {
    object Idle : UiState()
    object Loading : UiState()
    data class Success(val data: ...) : UiState()
    data class Error(val message: String?) : UiState()
}
```

## Qué NO vive aquí

- Lógica de negocio → va en `domain/service/` o `application/service/`
- Acceso a base de datos, red o sensores → va en `infrastructure/`
- Definición de contratos de I/O → va en `application/port/`

## Relación con otras capas

```
presentation ──► application/port/in  (ViewModels invocan use cases)
presentation ──► domain/model         (usa modelos para renderizar UI)
presentation ✗── infrastructure       (prohibido: no importar clases concretas de infra)
```
