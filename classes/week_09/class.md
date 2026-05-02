# Semana 9 — Bases de Datos Locales: Room, DataStore y WorkManager
**Curso:** Desarrollo de Sistemas Móviles | **Unidad:** U3  
**Sesión:** Presencial | **Duración:** 2 horas

---

## Objetivos de la Sesión

- Implementar Room 2.6+ para persistencia local con consultas reactivas.
- Reemplazar SharedPreferences con DataStore (Preferences y Proto).
- Programar tareas en segundo plano con WorkManager 2.9+.
- Comprender las diferencias entre Firebase Realtime Database y Firestore.
- Implementar Firebase Authentication para gestión de usuarios.

---

## 1. Room 2.6+ — Base de Datos Local *(35 min)*

### 1.1 ¿Por qué Room en lugar de SQLite directo?

| Aspecto | SQLite directo | Room |
|---|---|---|
| Consultas | Strings de SQL — sin verificación | Verificadas en compilación |
| Resultados | Cursor manual | Data classes automáticas |
| Reactividad | Sin soporte | Flow<List<T>> nativo |
| Migraciones | Manual y propenso a errores | Automation con scripts versionados |
| Testing | Complejo | Fácil con in-memory database |
| Tipo de datos | Solo tipos SQL básicos | Converters para tipos complejos |

### 1.2 La tríada de Room: Entity, DAO, Database

```kotlin
// 1. ENTITY — representa una tabla en la BD
@Entity(tableName = "products")
data class ProductEntity(
    @PrimaryKey val id: Int,
    @ColumnInfo(name = "product_name") val nombre: String,
    val precio: Double,
    val categoria: String,
    val disponible: Boolean = true,
    @ColumnInfo(name = "created_at") val creadoEn: Long = System.currentTimeMillis()
)

// 2. DAO — Data Access Object: interfaz de consultas
@Dao
interface ProductDao {
    // Flow<List>: reactividad — la UI se actualiza automáticamente cuando cambian los datos
    @Query("SELECT * FROM products ORDER BY product_name ASC")
    fun getAll(): Flow<List<ProductEntity>>

    @Query("SELECT * FROM products WHERE id = :id")
    suspend fun getById(id: Int): ProductEntity?

    @Query("SELECT * FROM products WHERE categoria = :categoria")
    fun getByCategoria(categoria: String): Flow<List<ProductEntity>>

    @Query("SELECT * FROM products WHERE product_name LIKE '%' || :query || '%'")
    fun search(query: String): Flow<List<ProductEntity>>

    // INSERT OR REPLACE: inserta si no existe, actualiza si existe (upsert)
    @Upsert
    suspend fun upsert(product: ProductEntity)

    @Upsert
    suspend fun upsertAll(products: List<ProductEntity>)

    @Delete
    suspend fun delete(product: ProductEntity)

    @Query("DELETE FROM products WHERE id = :id")
    suspend fun deleteById(id: Int)

    @Query("SELECT COUNT(*) FROM products")
    suspend fun count(): Int
}

// 3. DATABASE — el punto de entrada a Room
@Database(
    entities = [ProductEntity::class],
    version = 1,
    exportSchema = true  // exportar esquema para control de versiones
)
@TypeConverters(Converters::class)
abstract class AppDatabase : RoomDatabase() {
    abstract fun productDao(): ProductDao

    companion object {
        @Volatile
        private var INSTANCE: AppDatabase? = null

        fun getInstance(context: Context): AppDatabase {
            return INSTANCE ?: synchronized(this) {
                Room.databaseBuilder(
                    context.applicationContext,
                    AppDatabase::class.java,
                    "app_database"
                )
                .fallbackToDestructiveMigration()  // solo en desarrollo
                .build()
                .also { INSTANCE = it }
            }
        }
    }
}

// TypeConverter — para tipos no soportados nativamente por Room
class Converters {
    @TypeConverter
    fun fromList(value: List<String>): String = value.joinToString(",")

    @TypeConverter
    fun toList(value: String): List<String> = value.split(",")
}
```

### 1.3 Migraciones — nunca destructive en producción

```kotlin
// Cuando cambias el esquema, SIEMPRE escribe una migración
val MIGRATION_1_2 = object : Migration(1, 2) {
    override fun migrate(db: SupportSQLiteDatabase) {
        db.execSQL("ALTER TABLE products ADD COLUMN stock INTEGER NOT NULL DEFAULT 0")
    }
}

// Registrar la migración en la Database
Room.databaseBuilder(context, AppDatabase::class.java, "app_database")
    .addMigrations(MIGRATION_1_2)
    .build()
```

### 1.4 Room con Hilt

```kotlin
// di/DatabaseModule.kt
@Module
@InstallIn(SingletonComponent::class)
object DatabaseModule {

    @Provides
    @Singleton
    fun provideDatabase(@ApplicationContext context: Context): AppDatabase =
        Room.databaseBuilder(context, AppDatabase::class.java, "app_database")
            .build()

    @Provides
    fun provideProductDao(database: AppDatabase): ProductDao =
        database.productDao()
}
```

---

## 2. DataStore — Reemplazo de SharedPreferences *(20 min)*

### 2.1 Problemas con SharedPreferences

SharedPreferences tiene problemas serios en producción:
- Las operaciones de lectura/escritura pueden bloquear el hilo principal.
- No es null-safe — puede retornar null inesperadamente.
- No tiene soporte para coroutines ni Flow.
- No verifica tipos en compilación.

**Solución: Jetpack DataStore** (Flow-based, type-safe, no bloqueante).

### 2.2 Preferences DataStore (clave-valor simple)

```kotlin
// Definir las claves de preferencias
object PreferenciasKeys {
    val TEMA_OSCURO = booleanPreferencesKey("tema_oscuro")
    val IDIOMA = stringPreferencesKey("idioma")
    val PRIMERA_VEZ = booleanPreferencesKey("primera_vez")
    val ID_USUARIO = stringPreferencesKey("id_usuario")
}

// DataStore manager
class PreferenciasRepository @Inject constructor(
    private val dataStore: DataStore<Preferences>
) {
    // Leer una preferencia — retorna Flow (reactivo)
    val temaOscuro: Flow<Boolean> = dataStore.data
        .catch { emit(emptyPreferences()) }  // manejar corrupción del archivo
        .map { preferences -> preferences[PreferenciasKeys.TEMA_OSCURO] ?: false }

    val idioma: Flow<String> = dataStore.data
        .map { it[PreferenciasKeys.IDIOMA] ?: "es" }

    // Guardar una preferencia
    suspend fun setTemaOscuro(oscuro: Boolean) {
        dataStore.edit { preferences ->
            preferences[PreferenciasKeys.TEMA_OSCURO] = oscuro
        }
    }

    suspend fun setIdioma(idioma: String) {
        dataStore.edit { it[PreferenciasKeys.IDIOMA] = idioma }
    }
}

// Configurar en Hilt
@Module
@InstallIn(SingletonComponent::class)
object DataStoreModule {
    @Provides
    @Singleton
    fun provideDataStore(@ApplicationContext context: Context): DataStore<Preferences> =
        PreferenceDataStoreFactory.create(
            produceFile = { context.preferencesDataStoreFile("user_preferences") }
        )
}
```

---

## 3. WorkManager 2.9+ — Tareas en Segundo Plano *(20 min)*

### 3.1 ¿Cuándo usar WorkManager?

WorkManager es para tareas que **deben completarse eventualmente**, incluso si la app se cierra o el dispositivo se reinicia:
- Sincronizar datos en background.
- Subir fotos a la nube.
- Enviar logs de analytics.
- Limpiar datos temporales.

NO usar para tareas inmediatas o de tiempo real — para eso usa coroutines en el ViewModel.

### 3.2 Implementación

```kotlin
// Un Worker define el trabajo a realizar
class SincronizacionWorker(
    context: Context,
    workerParams: WorkerParameters
) : CoroutineWorker(context, workerParams) {

    override suspend fun doWork(): Result {
        return try {
            // Aquí va el trabajo (corre en Dispatchers.IO automáticamente)
            val datos = api.obtenerActualizaciones()
            database.guardarTodos(datos)
            Result.success()
        } catch (e: Exception) {
            if (runAttemptCount < 3) {
                Result.retry()  // reintentar hasta 3 veces
            } else {
                Result.failure()
            }
        }
    }
}

// Programar el trabajo
val workRequest = PeriodicWorkRequestBuilder<SincronizacionWorker>(
    repeatInterval = 6,
    repeatIntervalTimeUnit = TimeUnit.HOURS
)
.setConstraints(
    Constraints.Builder()
        .setRequiredNetworkType(NetworkType.CONNECTED)
        .setRequiresBatteryNotLow(true)
        .build()
)
.build()

WorkManager.getInstance(context).enqueueUniquePeriodicWork(
    "sincronizacion_periodica",
    ExistingPeriodicWorkPolicy.KEEP,  // no reemplazar si ya existe
    workRequest
)
```

---

## 4. Firebase Authentication *(20 min)*

### 4.1 Métodos de autenticación disponibles

| Método | Cuándo usarlo |
|---|---|
| Email + Contraseña | Apps propias sin SSO requerido |
| Google Sign-In | Mayor UX — un clic, sin contraseñas |
| Autenticación anónima | Permitir uso sin registro (luego vincular cuenta) |
| Phone (SMS OTP) | Verificación por número de celular |
| Apple Sign-In | Obligatorio en iOS si ofreces Google Sign-In |

### 4.2 Implementación de Email/Contraseña

```kotlin
class AuthRepositoryImpl @Inject constructor() : AuthRepository {
    private val auth = Firebase.auth

    // Estado del usuario actual como Flow
    val currentUser: Flow<FirebaseUser?> = callbackFlow {
        val listener = FirebaseAuth.AuthStateListener { trySend(it.currentUser) }
        auth.addAuthStateListener(listener)
        awaitClose { auth.removeAuthStateListener(listener) }
    }

    suspend fun signIn(email: String, password: String): Result<FirebaseUser> =
        runCatching {
            auth.signInWithEmailAndPassword(email, password).await().user
                ?: throw Exception("Error de autenticación")
        }

    suspend fun signUp(email: String, password: String): Result<FirebaseUser> =
        runCatching {
            auth.createUserWithEmailAndPassword(email, password).await().user
                ?: throw Exception("No se pudo crear el usuario")
        }

    fun signOut() = auth.signOut()

    fun getCurrentUserId(): String? = auth.currentUser?.uid
}
```

### 4.3 Firebase Security Rules — proteger datos

```javascript
// firestore.rules — solo el usuario autenticado puede acceder a sus propios datos
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    // Cada usuario solo puede leer/escribir su propia colección de tareas
    match /users/{userId}/tareas/{tareaId} {
      allow read, write: if request.auth != null && request.auth.uid == userId;
    }
    
    // Productos: cualquier autenticado puede leer, solo admins escriben
    match /productos/{productoId} {
      allow read: if request.auth != null;
      allow write: if request.auth != null && request.auth.token.admin == true;
    }
  }
}
```

---

## 5. Firestore vs Firebase Realtime Database

| Característica | Firestore | Realtime Database |
|---|---|---|
| Estructura | Documentos + colecciones (flexible) | Árbol JSON (rígido) |
| Consultas | Complejas (filtros, ordenamiento, paginación) | Básicas (solo por clave/valor) |
| Offline support | Completo (cache automática) | Básico |
| Escalabilidad | Automática (serverless) | Limitada |
| Precio | Por lectura/escritura/almacenamiento | Por GB transferido |
| **Recomendación 2026** | **✅ Primario** | ❌ Legacy — solo para proyectos existentes |

---

## 6. Resumen de la Sesión

| Concepto | Clave a recordar |
|---|---|
| Room | Entity + DAO + Database. `Flow<List<T>>` para reactividad. `@Upsert` para insert-or-update. |
| Migraciones | Siempre escribir `Migration(oldVer, newVer)` en producción — nunca destructiva. |
| DataStore | Reemplaza SharedPreferences. `Flow<T>` para leer, `suspend dataStore.edit {}` para escribir. |
| WorkManager | Para tareas diferibles que sobreviven la muerte de la app. Constraints de red y batería. |
| Firebase Auth | `callbackFlow` para el estado del usuario como Flow reactivo. |
| Security Rules | Siempre proteger con `request.auth.uid == userId`. Nunca dejar reglas abiertas. |

---

## Recursos Adicionales

| Recurso | Enlace | Para qué |
|---|---|---|
| Room Docs | [developer.android.com/training/data-storage/room](https://developer.android.com/training/data-storage/room) | Guía oficial Room |
| DataStore Docs | [developer.android.com/topic/libraries/architecture/datastore](https://developer.android.com/topic/libraries/architecture/datastore) | Guía oficial DataStore |
| WorkManager | [developer.android.com/topic/libraries/architecture/workmanager](https://developer.android.com/topic/libraries/architecture/workmanager) | Guía oficial WorkManager |
| Firebase Auth | [firebase.google.com/docs/auth/android/start](https://firebase.google.com/docs/auth/android/start) | Autenticación Firebase |
| Firestore Security Rules | [firebase.google.com/docs/firestore/security/get-started](https://firebase.google.com/docs/firestore/security/get-started) | Proteger datos Firestore |
