# Infrastructure Layer

**Regla: puede importar `application` y `domain`. No puede importar `presentation`.**

> **Nota de naming**: el paquete actualmente se llama `infraestructure` (sin la segunda `r`).
> El nombre correcto en inglés es `infrastructure`. Corregir en una refactorización futura.

Esta capa es el adaptador de salida. Implementa los contratos definidos en `application/port/out/`
usando tecnologías concretas: Room, Retrofit, Firebase, CameraX, sensores Android.

## Responsabilidad

- Conectar el sistema con el mundo exterior (red, base de datos local, dispositivo)
- Traducir entre modelos de dominio y estructuras propias del framework (Entities, DTOs)
- Proveer los módulos Hilt que ligan interfaces con implementaciones

## Estructura

```
infrastructure/
├── db/                         # Room
│   ├── AppDatabase.kt
│   ├── dao/                    # Interfaces DAO
│   ├── entity/                 # @Entity data classes (NO son domain models)
│   └── mapper/                 # Conversión entity ↔ domain model
├── network/                    # Retrofit
│   ├── ApiService.kt           # @GET, @POST interfaces
│   ├── dto/                    # Data Transfer Objects
│   └── mapper/                 # Conversión DTO ↔ domain model
├── di/                         # Módulos Hilt
│   └── AppModule.kt            # Bindea port/out con sus implementaciones
└── adapters/                   # Adaptadores de dispositivo
    ├── camera/                 # CameraX
    ├── sensor/                 # Acelerómetro, podómetro
    └── location/               # FusedLocationProvider
```

## Mappers

**Los mappers son obligatorios.** Nunca se deben usar `@Entity` de Room ni DTOs de Retrofit
directamente en la capa de `application` o `domain`. El mapper convierte en ambas direcciones:

```kotlin
fun ClinicEntity.toDomain(): Clinic = Clinic(id = this.id, name = this.name, ...)
fun Clinic.toEntity(): ClinicEntity = ClinicEntity(id = this.id, name = this.name, ...)
```

## Implementación de Output Ports

Cada clase en `infrastructure` que implementa un `port/out` debe declararlo explícitamente:

```kotlin
class UserRepositoryImpl @Inject constructor(
    private val dao: UserDao,
    private val preferences: SessionPreferences,
) : UserRepository {               // ← implementa application/port/out
    override fun getUser(): Flow<User?> = dao.getUser().map { it?.toDomain() }
    override suspend fun saveUser(user: User) = dao.insert(user.toEntity())
    override suspend fun clearUser() = dao.deleteAll()
}
```

## Módulos Hilt

Los módulos Hilt son el único lugar donde se menciona tanto la interfaz (`port/out`) como la
implementación concreta:

```kotlin
@Module
@InstallIn(SingletonComponent::class)
abstract class RepositoryModule {
    @Binds
    abstract fun bindUserRepository(impl: UserRepositoryImpl): UserRepository
}
```

## Qué NO vive aquí

- Lógica de negocio → va en `domain/service/`
- Definición de interfaces de repositorios → va en `application/port/out/`
- Composables o ViewModels → van en `presentation/`

## Relación con otras capas

```
infrastructure ──► application/port/out  (implementa contratos de I/O)
infrastructure ──► domain/model          (usa modelos para mappers)
infrastructure ✗── presentation          (prohibido)
application    ──► infrastructure        (prohibido en sentido inverso: application no importa infra)
```
