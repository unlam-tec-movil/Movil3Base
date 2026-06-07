# Domain Layer

**Regla de oro: cero importaciones de Android, Room, Retrofit, Hilt o cualquier framework externo.**

Esta capa representa el conocimiento de negocio puro. Es el núcleo del hexágono. Todo lo demás
depende de ella; ella no depende de nadie.

## Qué vive aquí

### `model/`

Modelos de dominio representados como `data class` de Kotlin. Describen los conceptos centrales
del negocio (ej. `User`, `Session`, `Exercise`, `Clinic`).

- Sin anotaciones de Room (`@Entity`, `@ColumnInfo`, etc.)
- Sin lógica de serialización
- Sin referencias a Android

### `service/`

Servicios de dominio: funciones puras y stateless que operan sobre los modelos.

- Reciben modelos de dominio, devuelven modelos de dominio o tipos primitivos
- No hacen I/O (no acceden a DB, red, sensores)
- Son fácilmente testeables con JUnit puro (sin Robolectric, sin emulador)

Ejemplo:

```kotlin
class AngleComparisonService @Inject constructor() {
    fun compare(measured: Float, target: Float): Feedback {
        val delta = abs(measured - target)
        return when {
            delta <= 15f -> Feedback.IDEAL
            delta <= 30f -> Feedback.WARNING
            else -> Feedback.ERROR
        }
    }
}
```

## Qué NO vive aquí

- Interfaces de repositorios o fuentes de datos → van en `application/port/out/`
- Interfaces de casos de uso → van en `application/port/in/`
- Implementaciones de acceso a datos → van en `infrastructure/`
- Cualquier clase que importe `android.*`, `androidx.*` o frameworks externos

## Relación con otras capas

```
domain  <──── application (importa domain)
domain  <──── infrastructure (importa domain)
domain  <──── presentation (importa domain)
```

`domain` no importa ninguna de las otras tres capas.
