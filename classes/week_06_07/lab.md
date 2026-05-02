# CodLab N°6 — REST API con Retrofit + Firebase Firestore
**Curso:** Desarrollo de Sistemas Móviles | **Semanas:** 6–7  
**Evaluación:** Individual

---

## Objetivos

- Consumir una API REST pública con Retrofit + Kotlinx Serialization.
- Manejar estados de carga/éxito/error con coroutines y StateFlow.
- Integrar Firebase Firestore con listeners en tiempo real.
- Implementar operaciones CRUD en Firestore.
- Agregar OkHttp logging interceptor para depuración.

## Prerrequisitos

- CodLab N°4 completado (MVVM + Hilt).
- Cuenta en Firebase Console: [console.firebase.google.com](https://console.firebase.google.com).

---

## SESIÓN 1 (Semana 6) — Retrofit + API Pública

### Paso 1: Dependencias

```kotlin
// app/build.gradle.kts
dependencies {
    // Retrofit
    implementation("com.squareup.retrofit2:retrofit:2.11.0")
    implementation("com.jakewharton.retrofit2:retrofit2-kotlinx-serialization-converter:1.0.0")

    // OkHttp
    implementation("com.squareup.okhttp3:okhttp:4.12.0")
    implementation("com.squareup.okhttp3:logging-interceptor:4.12.0")

    // Kotlinx Serialization
    implementation("org.jetbrains.kotlinx:kotlinx-serialization-json:1.7.3")

    // Coil para imágenes
    implementation("io.coil-kt.coil3:coil-compose:3.0.4")
    implementation("io.coil-kt.coil3:coil-network-okhttp:3.0.4")
}
```

### Paso 2: API — JSONPlaceholder

Usaremos [jsonplaceholder.typicode.com](https://jsonplaceholder.typicode.com) (API pública, sin clave):

```kotlin
// data/remote/model/PostDto.kt
@Serializable
data class PostDto(
    val id: Int,
    val userId: Int,
    val title: String,
    val body: String
)

// data/remote/api/PostsApi.kt
interface PostsApi {
    @GET("posts")
    suspend fun getPosts(): List<PostDto>

    @GET("posts/{id}")
    suspend fun getPost(@Path("id") id: Int): PostDto

    @GET("posts")
    suspend fun getPostsByUser(@Query("userId") userId: Int): List<PostDto>
}
```

### Paso 3: Módulo Hilt para red

```kotlin
// di/NetworkModule.kt
@Module
@InstallIn(SingletonComponent::class)
object NetworkModule {

    @Provides
    @Singleton
    fun provideOkHttpClient(): OkHttpClient =
        OkHttpClient.Builder()
            .addInterceptor(HttpLoggingInterceptor().apply {
                level = if (BuildConfig.DEBUG) HttpLoggingInterceptor.Level.BODY
                        else HttpLoggingInterceptor.Level.NONE
            })
            .connectTimeout(30, TimeUnit.SECONDS)
            .readTimeout(30, TimeUnit.SECONDS)
            .build()

    @Provides
    @Singleton
    fun provideJson(): Json = Json {
        ignoreUnknownKeys = true  // ignorar campos del JSON que no están en el modelo
        coerceInputValues = true  // usar valor por defecto si el tipo no coincide
    }

    @Provides
    @Singleton
    fun provideRetrofit(okHttpClient: OkHttpClient, json: Json): Retrofit =
        Retrofit.Builder()
            .baseUrl("https://jsonplaceholder.typicode.com/")
            .client(okHttpClient)
            .addConverterFactory(json.asConverterFactory("application/json".toMediaType()))
            .build()

    @Provides
    @Singleton
    fun providePostsApi(retrofit: Retrofit): PostsApi =
        retrofit.create(PostsApi::class.java)
}
```

### Paso 4: Repository y ViewModel

```kotlin
// data/repository/PostsRepository.kt
interface PostsRepository {
    suspend fun getPosts(): Result<List<PostDto>>
    suspend fun getPost(id: Int): Result<PostDto>
}

class PostsRepositoryImpl @Inject constructor(
    private val api: PostsApi
) : PostsRepository {

    override suspend fun getPosts(): Result<List<PostDto>> =
        runCatching { api.getPosts() }

    override suspend fun getPost(id: Int): Result<PostDto> =
        runCatching { api.getPost(id) }
}

// ui/PostsViewModel.kt
@HiltViewModel
class PostsViewModel @Inject constructor(
    private val repository: PostsRepository
) : ViewModel() {

    data class UiState(
        val posts: List<PostDto> = emptyList(),
        val isLoading: Boolean = false,
        val error: String? = null
    )

    private val _uiState = MutableStateFlow(UiState())
    val uiState: StateFlow<UiState> = _uiState.asStateFlow()

    init { cargarPosts() }

    fun cargarPosts() {
        viewModelScope.launch {
            _uiState.update { it.copy(isLoading = true, error = null) }
            repository.getPosts()
                .onSuccess { posts ->
                    _uiState.update { it.copy(posts = posts, isLoading = false) }
                }
                .onFailure { error ->
                    _uiState.update { it.copy(error = error.message, isLoading = false) }
                }
        }
    }
}
```

### Paso 5: Screen Compose

```kotlin
@Composable
fun PostsScreen(
    onPostClick: (Int) -> Unit = {},
    viewModel: PostsViewModel = hiltViewModel()
) {
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()

    Scaffold(topBar = { TopAppBar(title = { Text("Posts") }) }) { padding ->
        Box(Modifier.fillMaxSize().padding(padding)) {
            when {
                uiState.isLoading -> CircularProgressIndicator(Modifier.align(Alignment.Center))
                uiState.error != null -> Column(Modifier.align(Alignment.Center), horizontalAlignment = Alignment.CenterHorizontally) {
                    Text("Error: ${uiState.error}")
                    Button(onClick = { viewModel.cargarPosts() }) { Text("Reintentar") }
                }
                else -> LazyColumn(contentPadding = PaddingValues(16.dp), verticalArrangement = Arrangement.spacedBy(8.dp)) {
                    items(uiState.posts, key = { it.id }) { post ->
                        PostCard(post = post, onClick = { onPostClick(post.id) })
                    }
                }
            }
        }
    }
}

@Composable
fun PostCard(post: PostDto, onClick: () -> Unit) {
    Card(onClick = onClick, modifier = Modifier.fillMaxWidth()) {
        Column(Modifier.padding(16.dp)) {
            Text(post.title, style = MaterialTheme.typography.titleSmall, maxLines = 2)
            Spacer(Modifier.height(4.dp))
            Text(post.body, style = MaterialTheme.typography.bodySmall, maxLines = 2)
        }
    }
}
```

---

## SESIÓN 2 (Semana 7) — Firebase Firestore

### Paso 6: Configurar Firebase

1. Ve a [console.firebase.google.com](https://console.firebase.google.com).
2. Crea un nuevo proyecto (o usa uno existente).
3. Agrega una app Android con tu package name.
4. Descarga `google-services.json` → colócalo en `app/`.
5. Agrega los plugins y dependencias:

```kotlin
// build.gradle.kts (proyecto raíz)
plugins {
    id("com.google.gms.google-services") version "4.4.2" apply false
}

// app/build.gradle.kts
plugins {
    id("com.google.gms.google-services")
}
dependencies {
    implementation(platform("com.google.firebase:firebase-bom:33.0.0"))
    implementation("com.google.firebase:firebase-firestore")
    implementation("com.google.firebase:firebase-auth")
    // Extensiones Kotlin para usar .await() en coroutines
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-play-services:1.8.1")
}
```

### Paso 7: Modelo y repositorio Firestore

```kotlin
// data/remote/model/TareaFirestore.kt
@Serializable
data class TareaFirestore(
    val id: String = "",
    val titulo: String = "",
    val descripcion: String = "",
    val completada: Boolean = false,
    val timestamp: Long = System.currentTimeMillis()
)

// data/repository/TareasFirestoreRepository.kt
class TareasFirestoreRepository @Inject constructor() {
    private val db = Firebase.firestore
    private val coleccion = db.collection("tareas")

    fun getTareasFlow(): Flow<List<TareaFirestore>> = callbackFlow {
        val listener = coleccion
            .orderBy("timestamp", Query.Direction.DESCENDING)
            .addSnapshotListener { snapshot, error ->
                if (error != null) { close(error); return@addSnapshotListener }
                val tareas = snapshot?.toObjects(TareaFirestore::class.java) ?: emptyList()
                trySend(tareas)
            }
        awaitClose { listener.remove() }
    }

    suspend fun agregarTarea(tarea: TareaFirestore) {
        coleccion.document(UUID.randomUUID().toString()).set(tarea).await()
    }

    suspend fun toggleCompletar(id: String, completada: Boolean) {
        coleccion.document(id).update("completada", !completada).await()
    }

    suspend fun eliminarTarea(id: String) {
        coleccion.document(id).delete().await()
    }
}
```

### Paso 8: ViewModel con Firestore

```kotlin
@HiltViewModel
class TareasFirestoreViewModel @Inject constructor(
    private val repository: TareasFirestoreRepository
) : ViewModel() {

    val tareas: StateFlow<List<TareaFirestore>> = repository.getTareasFlow()
        .stateIn(
            scope = viewModelScope,
            started = SharingStarted.WhileSubscribed(5000),
            initialValue = emptyList()
        )

    fun agregarTarea(titulo: String, descripcion: String) {
        viewModelScope.launch {
            repository.agregarTarea(TareaFirestore(titulo = titulo, descripcion = descripcion))
        }
    }

    fun toggleCompletar(id: String, completada: Boolean) {
        viewModelScope.launch { repository.toggleCompletar(id, completada) }
    }

    fun eliminarTarea(id: String) {
        viewModelScope.launch { repository.eliminarTarea(id) }
    }
}
```

### Paso 9: UI de Firestore en tiempo real

```kotlin
@Composable
fun TareasFirestoreScreen(viewModel: TareasFirestoreViewModel = hiltViewModel()) {
    val tareas by viewModel.tareas.collectAsStateWithLifecycle()
    var showDialog by remember { mutableStateOf(false) }

    Scaffold(
        topBar = { TopAppBar(title = { Text("Tareas (Firestore)") }) },
        floatingActionButton = {
            FloatingActionButton(onClick = { showDialog = true }) {
                Icon(Icons.Filled.Add, contentDescription = "Agregar tarea")
            }
        }
    ) { padding ->
        LazyColumn(
            contentPadding = padding,
            verticalArrangement = Arrangement.spacedBy(8.dp),
            modifier = Modifier.padding(horizontal = 16.dp)
        ) {
            items(tareas, key = { it.id }) { tarea ->
                SwipeToDismissBox(
                    state = rememberSwipeToDismissBoxState(
                        confirmValueChange = { value ->
                            if (value == SwipeToDismissBoxValue.EndToStart) {
                                viewModel.eliminarTarea(tarea.id)
                            }
                            true
                        }
                    ),
                    backgroundContent = {
                        Box(Modifier.fillMaxSize().background(MaterialTheme.colorScheme.errorContainer)) {
                            Icon(Icons.Filled.Delete, contentDescription = null,
                                modifier = Modifier.align(Alignment.CenterEnd).padding(16.dp))
                        }
                    }
                ) {
                    Card(modifier = Modifier.fillMaxWidth()) {
                        Row(Modifier.padding(16.dp), verticalAlignment = Alignment.CenterVertically) {
                            Checkbox(
                                checked = tarea.completada,
                                onCheckedChange = { viewModel.toggleCompletar(tarea.id, tarea.completada) }
                            )
                            Column(Modifier.weight(1f)) {
                                Text(tarea.titulo, style = MaterialTheme.typography.titleSmall)
                                Text(tarea.descripcion, style = MaterialTheme.typography.bodySmall)
                            }
                        }
                    }
                }
            }
        }

        if (showDialog) {
            AgregarTareaDialog(
                onConfirm = { titulo, desc ->
                    viewModel.agregarTarea(titulo, desc)
                    showDialog = false
                },
                onDismiss = { showDialog = false }
            )
        }
    }
}
```

---

## Entregables

**Sesión 1:**
- App consumiendo la API de JSONPlaceholder mostrando lista de posts.
- Estados Loading/Success/Error correctamente manejados.
- OkHttp logging visible en Logcat (tag: `OkHttp`).

**Sesión 2:**
- Firebase configurado con `google-services.json`.
- Lista de tareas en tiempo real desde Firestore.
- Agregar nueva tarea funcional (con dialog o bottom sheet).
- Eliminar tarea con swipe.
- Marcar tarea como completada.

## Criterios de Evaluación

| Criterio | Puntaje |
|---|---|
| Retrofit configurado con Kotlinx Serialization (no Gson) | 2 pts |
| OkHttp logging interceptor funcionando | 1 pt |
| Estados Loading/Success/Error correctamente manejados | 2 pts |
| Firebase Firestore con listener en tiempo real (no one-shot) | 2 pts |
| CRUD en Firestore: crear + toggle + eliminar | 2 pts |
| Manejo de errores (no crash si se pierde la conexión) | 1 pt |
| **Total** | **10 pts** |
