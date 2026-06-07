# Data Layer (Outbound Adapter)

**Regla: puede importar `application` y `domain`. No puede importar `ui`.**

Esta capa es el adaptador de salida. Implementa los contratos definidos en `application/port/out/`
usando tecnologías concretas: Room, Retrofit, Firebase, CameraX, sensores Android.

## Responsabilidad

- Conectar el sistema con el mundo exterior (red, base de datos local, dispositivo)
- Traducir entre modelos de dominio y estructuras propias del framework (Entities, DTOs)
- Proveer los módulos Hilt que ligan interfaces con implementaciones

## Estructura

```
data/
├── datasources/
│   ├── local/                  # Room
│   │   ├── database/           # AppDatabase.kt
│   │   ├── dao/                # Interfaces DAO
│   │   └── entities/           # @Entity data classes (NO son domain models)
│   └── network/                # Retrofit
│       ├── ApiService.kt       # @GET, @POST interfaces
│       └── dto/                # Data Transfer Objects
├── mappers/                    # Conversión entity/DTO ↔ domain model
├── repositories/               # Implementaciones de application/port/out
└── di/                         # Módulos Hilt
```

## Mappers

**Los mappers son obligatorios.** Nunca se deben usar `@Entity` de Room ni DTOs de Retrofit
directamente en la capa de `application` o `domain`. El mapper convierte en ambas direcciones:

```kotlin
fun UserEntity.toDomain(): User = User(id = this.id, name = this.name, ...)
fun User.toEntity(): UserEntity = UserEntity(id = this.id, name = this.name, ...)
```

## Implementación de Output Ports

Cada repositorio en `data/repositories/` implementa explícitamente un `port/out`:

```kotlin
class UserRepositoryImpl @Inject constructor(
    private val dao: UserDao,
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
- Composables o ViewModels → van en `ui/`

## Relación con otras capas

```
data ──► application/port/out  (implementa contratos de I/O)
data ──► domain/model          (usa modelos para mappers)
data ✗── ui                    (prohibido)
application ✗── data           (prohibido en sentido inverso: application no importa data)
```

