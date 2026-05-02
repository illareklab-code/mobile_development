# CodLab N°8 — Room + DataStore + Firebase Authentication
**Curso:** Desarrollo de Sistemas Móviles | **Semana:** 9  
**Evaluación:** Individual

---

## Objetivos

- Implementar Room con consultas reactivas usando Flow.
- Usar DataStore para persistir preferencias de usuario.
- Implementar un flujo completo de autenticación con Firebase Auth.
- Proteger Firestore con Security Rules de usuario.
- Configurar Room con KSP (no KAPT).

## Prerrequisitos

- CodLab N°6 completado (Firebase configurado).
- Hilt funcionando en el proyecto.

---

## Paso 1: Dependencias Room + DataStore

```kotlin
// app/build.gradle.kts
plugins {
    id("com.google.devtools.ksp") version "2.0.21-1.0.25"
}

dependencies {
    // Room
    val roomVersion = "2.6.1"
    implementation("androidx.room:room-runtime:$roomVersion")
    implementation("androidx.room:room-ktx:$roomVersion")  // extensiones Kotlin + Flow
    ksp("androidx.room:room-compiler:$roomVersion")         // KSP (no KAPT)

    // DataStore
    implementation("androidx.datastore:datastore-preferences:1.1.1")

    // WorkManager
    implementation("androidx.work:work-runtime-ktx:2.9.1")
}
```

---

## Paso 2: Room — Entity y DAO

```kotlin
// data/local/entity/NotaEntity.kt
@Entity(tableName = "notas")
data class NotaEntity(
    @PrimaryKey(autoGenerate = true) val id: Int = 0,
    val titulo: String,
    val contenido: String,
    val color: String = "AMARILLO",
    val completada: Boolean = false,
    val creadaEn: Long = System.currentTimeMillis()
)

// data/local/dao/NotaDao.kt
@Dao
interface NotaDao {
    @Query("SELECT * FROM notas ORDER BY creadaEn DESC")
    fun getAll(): Flow<List<NotaEntity>>

    @Query("SELECT * FROM notas WHERE id = :id")
    suspend fun getById(id: Int): NotaEntity?

    @Query("SELECT * FROM notas WHERE titulo LIKE '%' || :query || '%' OR contenido LIKE '%' || :query || '%'")
    fun search(query: String): Flow<List<NotaEntity>>

    @Upsert
    suspend fun upsert(nota: NotaEntity): Long

    @Delete
    suspend fun delete(nota: NotaEntity)

    @Query("UPDATE notas SET completada = NOT completada WHERE id = :id")
    suspend fun toggleCompletar(id: Int)

    @Query("SELECT COUNT(*) FROM notas WHERE completada = 0")
    fun countPendientes(): Flow<Int>
}
```

---

## Paso 3: Database + Hilt module

```kotlin
// data/local/AppDatabase.kt
@Database(entities = [NotaEntity::class], version = 1, exportSchema = false)
abstract class AppDatabase : RoomDatabase() {
    abstract fun notaDao(): NotaDao
}

// di/DatabaseModule.kt
@Module
@InstallIn(SingletonComponent::class)
object DatabaseModule {

    @Provides
    @Singleton
    fun provideDatabase(@ApplicationContext context: Context): AppDatabase =
        Room.databaseBuilder(context, AppDatabase::class.java, "notas_db")
            .build()

    @Provides
    fun provideNotaDao(db: AppDatabase): NotaDao = db.notaDao()
}
```

---

## Paso 4: DataStore para preferencias

```kotlin
// data/preferences/UserPreferences.kt
data class UserPreferences(
    val temaOscuro: Boolean = false,
    val ordenarPorFecha: Boolean = true,
    val mostrarCompletadas: Boolean = true
)

object PrefsKeys {
    val TEMA_OSCURO = booleanPreferencesKey("tema_oscuro")
    val ORDENAR_POR_FECHA = booleanPreferencesKey("ordenar_fecha")
    val MOSTRAR_COMPLETADAS = booleanPreferencesKey("mostrar_completadas")
}

class UserPreferencesRepository @Inject constructor(
    private val dataStore: DataStore<Preferences>
) {
    val preferences: Flow<UserPreferences> = dataStore.data
        .catch { emit(emptyPreferences()) }
        .map { prefs ->
            UserPreferences(
                temaOscuro = prefs[PrefsKeys.TEMA_OSCURO] ?: false,
                ordenarPorFecha = prefs[PrefsKeys.ORDENAR_POR_FECHA] ?: true,
                mostrarCompletadas = prefs[PrefsKeys.MOSTRAR_COMPLETADAS] ?: true
            )
        }

    suspend fun setTemaOscuro(value: Boolean) {
        dataStore.edit { it[PrefsKeys.TEMA_OSCURO] = value }
    }

    suspend fun setMostrarCompletadas(value: Boolean) {
        dataStore.edit { it[PrefsKeys.MOSTRAR_COMPLETADAS] = value }
    }
}

// di/DataStoreModule.kt
@Module
@InstallIn(SingletonComponent::class)
object DataStoreModule {
    @Provides
    @Singleton
    fun provideDataStore(@ApplicationContext context: Context): DataStore<Preferences> =
        PreferenceDataStoreFactory.create(
            produceFile = { context.preferencesDataStoreFile("user_prefs") }
        )
}
```

---

## Paso 5: Firebase Auth — Flujo de Login

```kotlin
// data/auth/AuthRepository.kt
interface AuthRepository {
    val currentUser: Flow<FirebaseUser?>
    suspend fun signIn(email: String, password: String): Result<Unit>
    suspend fun signUp(email: String, password: String): Result<Unit>
    fun signOut()
    fun isSignedIn(): Boolean
}

class AuthRepositoryImpl @Inject constructor() : AuthRepository {
    private val auth = Firebase.auth

    override val currentUser: Flow<FirebaseUser?> = callbackFlow {
        val listener = FirebaseAuth.AuthStateListener { trySend(it.currentUser) }
        auth.addAuthStateListener(listener)
        awaitClose { auth.removeAuthStateListener(listener) }
    }

    override suspend fun signIn(email: String, password: String): Result<Unit> =
        runCatching { auth.signInWithEmailAndPassword(email, password).await(); Unit }

    override suspend fun signUp(email: String, password: String): Result<Unit> =
        runCatching { auth.createUserWithEmailAndPassword(email, password).await(); Unit }

    override fun signOut() = auth.signOut()
    override fun isSignedIn(): Boolean = auth.currentUser != null
}
```

---

## Paso 6: Pantalla de Login

```kotlin
@HiltViewModel
class LoginViewModel @Inject constructor(
    private val authRepository: AuthRepository
) : ViewModel() {

    data class LoginUiState(
        val email: String = "",
        val password: String = "",
        val isLoading: Boolean = false,
        val error: String? = null,
        val isLoggedIn: Boolean = false
    )

    private val _uiState = MutableStateFlow(LoginUiState())
    val uiState: StateFlow<LoginUiState> = _uiState.asStateFlow()

    init {
        viewModelScope.launch {
            authRepository.currentUser.collect { user ->
                _uiState.update { it.copy(isLoggedIn = user != null) }
            }
        }
    }

    fun onEmailChange(value: String) = _uiState.update { it.copy(email = value) }
    fun onPasswordChange(value: String) = _uiState.update { it.copy(password = value) }

    fun signIn() {
        viewModelScope.launch {
            _uiState.update { it.copy(isLoading = true, error = null) }
            authRepository.signIn(_uiState.value.email, _uiState.value.password)
                .onFailure { error -> _uiState.update { it.copy(error = error.message, isLoading = false) } }
                .onSuccess { _uiState.update { it.copy(isLoading = false) } }
        }
    }
}

@Composable
fun LoginScreen(
    onLoginSuccess: () -> Unit,
    viewModel: LoginViewModel = hiltViewModel()
) {
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()

    LaunchedEffect(uiState.isLoggedIn) {
        if (uiState.isLoggedIn) onLoginSuccess()
    }

    Column(
        Modifier.fillMaxSize().padding(32.dp),
        verticalArrangement = Arrangement.Center,
        horizontalAlignment = Alignment.CenterHorizontally
    ) {
        Text("Iniciar Sesión", style = MaterialTheme.typography.headlineMedium)
        Spacer(Modifier.height(32.dp))

        OutlinedTextField(
            value = uiState.email,
            onValueChange = viewModel::onEmailChange,
            label = { Text("Email") },
            keyboardOptions = KeyboardOptions(keyboardType = KeyboardType.Email),
            modifier = Modifier.fillMaxWidth()
        )
        Spacer(Modifier.height(8.dp))
        OutlinedTextField(
            value = uiState.password,
            onValueChange = viewModel::onPasswordChange,
            label = { Text("Contraseña") },
            visualTransformation = PasswordVisualTransformation(),
            modifier = Modifier.fillMaxWidth()
        )

        uiState.error?.let { error ->
            Spacer(Modifier.height(8.dp))
            Text(error, color = MaterialTheme.colorScheme.error)
        }

        Spacer(Modifier.height(16.dp))
        Button(
            onClick = viewModel::signIn,
            modifier = Modifier.fillMaxWidth(),
            enabled = !uiState.isLoading
        ) {
            if (uiState.isLoading) CircularProgressIndicator(Modifier.size(20.dp))
            else Text("Ingresar")
        }
    }
}
```

---

## Entregables

1. Room con al menos 1 entidad, DAO y queries reactivas con Flow.
2. DataStore persistiendo 2 preferencias (ej: tema oscuro + mostrar completadas).
3. Pantalla de login con Firebase Auth funcional.
4. Al cerrar y abrir la app, el usuario permanece logueado.
5. Firebase Security Rules configuradas para proteger datos por usuario.

## Criterios de Evaluación

| Criterio | Puntaje |
|---|---|
| Room configurado con KSP (no KAPT) y Flow funcionando | 3 pts |
| DataStore con 2 preferencias persistentes (verificar cerrando app) | 2 pts |
| Login/Logout de Firebase Auth funcional | 3 pts |
| Firebase Security Rules protegen datos por UID | 2 pts |
| **Total** | **10 pts** |
