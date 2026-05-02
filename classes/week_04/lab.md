# CodLab N°4 — MVVM + StateFlow + Hilt + Navegación
**Curso:** Desarrollo de Sistemas Móviles | **Semana:** 4  
**Evaluación:** Individual

---

## Objetivos

- Implementar MVVM con ViewModel y StateFlow.
- Configurar Hilt para inyección de dependencias.
- Crear navegación entre 2 pantallas con rutas type-safe.
- Demostrar Unidirectional Data Flow completo.
- Usar `collectAsStateWithLifecycle()` correctamente.

## Prerrequisitos

- CodLab N°3 completado.
- Leer la teoría de Semana 4.
- Dependencias de Hilt agregadas al proyecto.

---

## Paso 1: Configurar dependencias

En `build.gradle.kts` del módulo app:

```kotlin
plugins {
    id("com.google.dagger.hilt.android")
    id("com.google.devtools.ksp")
}

dependencies {
    // Hilt
    implementation("com.google.dagger:hilt-android:2.51.1")
    ksp("com.google.dagger:hilt-compiler:2.51.1")
    implementation("androidx.hilt:hilt-navigation-compose:1.2.0")

    // Navigation
    implementation("androidx.navigation:navigation-compose:2.7.7")

    // Lifecycle
    implementation("androidx.lifecycle:lifecycle-viewmodel-compose:2.8.7")
    implementation("androidx.lifecycle:lifecycle-runtime-compose:2.8.7")

    // Serialization (para rutas type-safe)
    implementation("org.jetbrains.kotlinx:kotlinx-serialization-json:1.7.3")
}
```

En `build.gradle.kts` del proyecto raíz:
```kotlin
plugins {
    id("com.google.dagger.hilt.android") version "2.51.1" apply false
}
```

---

## Paso 2: Application class

```kotlin
// app/src/main/kotlin/.../MiApp.kt
@HiltAndroidApp
class MiApp : Application()
```

Registra en `AndroidManifest.xml`:
```xml
<application android:name=".MiApp" ... >
```

---

## Paso 3: Modelo de datos y repositorio

```kotlin
// data/model/Nota.kt
data class Nota(
    val id: Int,
    val titulo: String,
    val contenido: String,
    val color: NoteColor = NoteColor.AMARILLO
)

enum class NoteColor(val hex: Long) {
    AMARILLO(0xFFFFF9C4),
    VERDE(0xFFC8E6C9),
    AZUL(0xFFBBDEFB),
    ROSA(0xFFF8BBD0)
}

// data/repository/NotasRepository.kt
interface NotasRepository {
    suspend fun getNotas(): List<Nota>
    suspend fun getNota(id: Int): Nota?
    suspend fun guardarNota(nota: Nota)
    suspend fun eliminarNota(id: Int)
}

// data/repository/NotasRepositoryImpl.kt
class NotasRepositoryImpl @Inject constructor() : NotasRepository {
    // En memoria por ahora (en semanas siguientes usaremos Room)
    private val notas = mutableListOf(
        Nota(1, "Compras", "Leche, pan, huevos", NoteColor.AMARILLO),
        Nota(2, "Proyecto DSM", "Implementar MVVM esta semana", NoteColor.AZUL),
        Nota(3, "Ideas app", "App de recetas con IA", NoteColor.VERDE)
    )

    override suspend fun getNotas(): List<Nota> = notas.toList()
    override suspend fun getNota(id: Int) = notas.find { it.id == id }
    override suspend fun guardarNota(nota: Nota) { notas.add(nota) }
    override suspend fun eliminarNota(id: Int) { notas.removeIf { it.id == id } }
}
```

---

## Paso 4: Módulo Hilt

```kotlin
// di/AppModule.kt
@Module
@InstallIn(SingletonComponent::class)
object AppModule {

    @Provides
    @Singleton
    fun provideNotasRepository(): NotasRepository = NotasRepositoryImpl()
}
```

---

## Paso 5: UseCases

```kotlin
// domain/GetNotasUseCase.kt
class GetNotasUseCase @Inject constructor(
    private val repository: NotasRepository
) {
    suspend operator fun invoke(): List<Nota> = repository.getNotas()
}

// domain/GetNotaUseCase.kt
class GetNotaUseCase @Inject constructor(
    private val repository: NotasRepository
) {
    suspend operator fun invoke(id: Int): Nota? = repository.getNota(id)
}
```

---

## Paso 6: ViewModels

```kotlin
// ui/lista/NotasListaUiState.kt
data class NotasListaUiState(
    val notas: List<Nota> = emptyList(),
    val isLoading: Boolean = false,
    val error: String? = null
)

// ui/lista/NotasListaViewModel.kt
@HiltViewModel
class NotasListaViewModel @Inject constructor(
    private val getNotasUseCase: GetNotasUseCase
) : ViewModel() {

    private val _uiState = MutableStateFlow(NotasListaUiState())
    val uiState: StateFlow<NotasListaUiState> = _uiState.asStateFlow()

    init { cargarNotas() }

    fun cargarNotas() {
        viewModelScope.launch {
            _uiState.update { it.copy(isLoading = true, error = null) }
            try {
                val notas = getNotasUseCase()
                _uiState.update { it.copy(notas = notas, isLoading = false) }
            } catch (e: Exception) {
                _uiState.update { it.copy(error = e.message, isLoading = false) }
            }
        }
    }
}

// ui/detalle/NotaDetalleViewModel.kt
@HiltViewModel
class NotaDetalleViewModel @Inject constructor(
    private val getNotaUseCase: GetNotaUseCase,
    savedStateHandle: SavedStateHandle
) : ViewModel() {

    private val notaId: Int = checkNotNull(savedStateHandle["notaId"])

    private val _uiState = MutableStateFlow<NotaDetalleUiState>(NotaDetalleUiState.Loading)
    val uiState: StateFlow<NotaDetalleUiState> = _uiState.asStateFlow()

    init { cargarNota() }

    private fun cargarNota() {
        viewModelScope.launch {
            val nota = getNotaUseCase(notaId)
            _uiState.value = if (nota != null) 
                NotaDetalleUiState.Success(nota)
            else 
                NotaDetalleUiState.Error("Nota no encontrada")
        }
    }
}

sealed class NotaDetalleUiState {
    data object Loading : NotaDetalleUiState()
    data class Success(val nota: Nota) : NotaDetalleUiState()
    data class Error(val message: String) : NotaDetalleUiState()
}
```

---

## Paso 7: Navegación type-safe

```kotlin
// navigation/AppRoutes.kt
@Serializable object RutaLista
@Serializable data class RutaDetalle(val notaId: Int)

// navigation/AppNavHost.kt
@Composable
fun AppNavHost() {
    val navController = rememberNavController()
    NavHost(navController = navController, startDestination = RutaLista) {
        composable<RutaLista> {
            NotasListaScreen(
                onNotaClick = { id -> navController.navigate(RutaDetalle(id)) }
            )
        }
        composable<RutaDetalle> {
            NotaDetalleScreen(onBack = navController::popBackStack)
        }
    }
}
```

---

## Paso 8: Screens Compose

```kotlin
// ui/lista/NotasListaScreen.kt
@Composable
fun NotasListaScreen(
    onNotaClick: (Int) -> Unit,
    viewModel: NotasListaViewModel = hiltViewModel()
) {
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()

    Scaffold(
        topBar = { TopAppBar(title = { Text("Mis Notas") }) }
    ) { padding ->
        Box(modifier = Modifier.fillMaxSize().padding(padding)) {
            when {
                uiState.isLoading -> CircularProgressIndicator(Modifier.align(Alignment.Center))
                uiState.error != null -> Text(
                    "Error: ${uiState.error}",
                    modifier = Modifier.align(Alignment.Center)
                )
                uiState.notas.isEmpty() -> Text(
                    "No hay notas. ¡Crea una!",
                    modifier = Modifier.align(Alignment.Center)
                )
                else -> LazyVerticalStaggeredGrid(
                    columns = StaggeredGridCells.Fixed(2),
                    contentPadding = PaddingValues(16.dp),
                    verticalItemSpacing = 8.dp,
                    horizontalArrangement = Arrangement.spacedBy(8.dp)
                ) {
                    items(uiState.notas, key = { it.id }) { nota ->
                        NotaCard(nota = nota, onClick = { onNotaClick(nota.id) })
                    }
                }
            }
        }
    }
}

@Composable
fun NotaCard(nota: Nota, onClick: () -> Unit) {
    Card(
        onClick = onClick,
        colors = CardDefaults.cardColors(
            containerColor = Color(nota.color.hex)
        )
    ) {
        Column(modifier = Modifier.padding(12.dp)) {
            Text(nota.titulo, style = MaterialTheme.typography.titleSmall)
            Spacer(Modifier.height(4.dp))
            Text(nota.contenido, style = MaterialTheme.typography.bodySmall, maxLines = 3)
        }
    }
}
```

---

## Entregables

1. App con 2 pantallas funcionales: lista de notas y detalle.
2. Navegación entre pantallas con type-safe routes.
3. ViewModel con StateFlow mostrando loading/success/error.
4. Hilt configurado: `@HiltAndroidApp`, `@HiltViewModel`, módulo con `@Provides`.
5. UseCase layer con al menos 2 use cases.

## Criterios de Evaluación

| Criterio | Puntaje |
|---|---|
| ViewModel con MutableStateFlow y UiState data class | 2 pts |
| `collectAsStateWithLifecycle()` en los screens | 1 pt |
| Hilt configurado correctamente (inyección funcional) | 2 pts |
| Navegación type-safe entre 2 pantallas | 2 pts |
| UseCase layer (mínimo 2 use cases) | 2 pts |
| Estado de loading visible en UI | 1 pt |
| **Total** | **10 pts** |
