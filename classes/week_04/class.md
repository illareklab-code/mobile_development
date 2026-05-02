# Semana 4 — Arquitectura MVVM, Navegación y Ciclo de Vida
**Curso:** Desarrollo de Sistemas Móviles | **Unidad:** U1  
**Sesión:** Presencial | **Duración:** 2 horas

---

## Objetivos de la Sesión

- Implementar la arquitectura MVVM con Unidirectional Data Flow (UDF).
- Usar StateFlow como mecanismo de estado reactivo en lugar de LiveData.
- Aplicar el patrón UseCase + Repository para separar responsabilidades.
- Configurar inyección de dependencias con Hilt.
- Implementar navegación type-safe con Jetpack Navigation 2.7+.

---

## 1. Ciclo de Vida de Android — Lo que Debes Saber *(15 min)*

### 1.1 Estados del ciclo de vida

```
Actividad creada
      ↓ onCreate()
   CREATED
      ↓ onStart()
    STARTED ← ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ┐
      ↓ onResume()                        │
   RESUMED (visible + interactiva)       │
      ↓ onPause()                        │
   PAUSED (parcialmente visible)         │
      ↓ onStop()                         │
   STOPPED (no visible)  ────────────── ─┘
      ↓ onDestroy()
  DESTROYED
```

**¿Por qué importa al desarrollador?**
- El SO puede **destruir** la Activity en cualquier momento (poca RAM, usuario sale de la app).
- Los datos en variables locales de la Activity **se pierden** al destruirse.
- El **ViewModel** surviva rotaciones de pantalla y cambios de configuración.

### 1.2 ViewModel — El puente entre datos y UI

```kotlin
// ❌ Mal: datos en la Activity → se pierden al rotar la pantalla
class MalaActivity : ComponentActivity() {
    private var contador = 0  // se resetea al rotar!
}

// ✅ Bien: datos en el ViewModel → sobrevive cambios de configuración
class BuenViewModel : ViewModel() {
    private val _contador = MutableStateFlow(0)
    val contador: StateFlow<Int> = _contador.asStateFlow()
}
```

---

## 2. MVVM + Unidirectional Data Flow (UDF) *(30 min)*

### 2.1 La arquitectura completa 2026

```
┌──────────────────────────────────────────────────────────┐
│                    UI LAYER                              │
│  ┌─────────────┐   ← estado ←   ┌──────────────────┐   │
│  │  Composable │                 │    ViewModel     │   │
│  │  (Screen)   │   → eventos →  │  (StateFlow)     │   │
│  └─────────────┘                 └──────────────────┘   │
└──────────────────────────────────────────────────────────┘
                              │ llama
┌──────────────────────────────────────────────────────────┐
│                   DOMAIN LAYER                           │
│              ┌──────────────────┐                        │
│              │    UseCase       │ ← 1 operación          │
│              │ GetProductsUseCase│                       │
│              └──────────────────┘                        │
└──────────────────────────────────────────────────────────┘
                              │ llama
┌──────────────────────────────────────────────────────────┐
│                    DATA LAYER                            │
│  ┌────────────────┐    ┌─────────────┐                  │
│  │  Repository    │ ←→ │ RemoteDS    │ (API REST)        │
│  │ (única fuente  │ ←→ │ LocalDS     │ (Room)            │
│  │  de verdad)    │    └─────────────┘                  │
│  └────────────────┘                                      │
└──────────────────────────────────────────────────────────┘
```

### 2.2 UiState — Modelar el estado de la pantalla

```kotlin
// Un UiState por pantalla — todo lo que la UI necesita en un solo objeto
data class ProductosUiState(
    val productos: List<Producto> = emptyList(),
    val isLoading: Boolean = false,
    val errorMessage: String? = null,
    val busqueda: String = ""
)
```

### 2.3 El ViewModel

```kotlin
@HiltViewModel
class ProductosViewModel @Inject constructor(
    private val getProductosUseCase: GetProductosUseCase
) : ViewModel() {

    private val _uiState = MutableStateFlow(ProductosUiState())
    val uiState: StateFlow<ProductosUiState> = _uiState.asStateFlow()

    init {
        cargarProductos()
    }

    fun cargarProductos() {
        viewModelScope.launch {
            _uiState.update { it.copy(isLoading = true) }
            try {
                val productos = getProductosUseCase()
                _uiState.update { it.copy(productos = productos, isLoading = false) }
            } catch (e: Exception) {
                _uiState.update { it.copy(errorMessage = e.message, isLoading = false) }
            }
        }
    }

    fun onBusquedaChange(texto: String) {
        _uiState.update { it.copy(busqueda = texto) }
    }

    fun onErrorDismissed() {
        _uiState.update { it.copy(errorMessage = null) }
    }
}
```

### 2.4 Conectar ViewModel a Compose

```kotlin
@Composable
fun ProductosScreen(
    viewModel: ProductosViewModel = hiltViewModel()
) {
    // collectAsStateWithLifecycle: para la colección cuando la app está en background
    // Más eficiente que collectAsState() — respeta el ciclo de vida
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()

    when {
        uiState.isLoading -> LoadingIndicator()
        uiState.errorMessage != null -> ErrorMessage(
            message = uiState.errorMessage!!,
            onDismiss = viewModel::onErrorDismissed
        )
        else -> ProductosList(
            productos = uiState.productos,
            busqueda = uiState.busqueda,
            onBusquedaChange = viewModel::onBusquedaChange
        )
    }
}
```

### 2.5 StateFlow vs LiveData — ¿Por qué migrar?

| Característica | LiveData | StateFlow |
|---|---|---|
| Requiere ciclo de vida | Sí (solo en Android) | No (Kotlin puro, multiplataforma) |
| Null safety | No (puede ser null) | Sí (siempre tiene valor inicial) |
| Transformaciones | limitado | `map`, `filter`, `combine`, `flatMapLatest` |
| Testing | Difícil (requiere Android) | Simple (Kotlin coroutines test) |
| Backpressure | No | Sí (conflation) |
| Multiplatforma (KMM) | No | ✅ Sí |

---

## 3. UseCase Layer *(15 min)*

Un UseCase es una clase con **una sola responsabilidad**: encapsular una operación de negocio.

```kotlin
// Cada UseCase = un archivo, una operación
class GetProductosUseCase @Inject constructor(
    private val productosRepository: ProductosRepository
) {
    // operator fun invoke() permite llamarlo como función: getProductosUseCase()
    suspend operator fun invoke(): List<Producto> {
        return productosRepository.getProductos()
            .filter { it.disponible }              // lógica de negocio
            .sortedBy { it.nombre }
    }
}

class BuscarProductosUseCase @Inject constructor(
    private val productosRepository: ProductosRepository
) {
    suspend operator fun invoke(query: String): List<Producto> {
        return productosRepository.buscarProductos(query)
    }
}
```

**¿Por qué no poner esta lógica en el ViewModel?**
- El ViewModel se volvería demasiado grande.
- Los UseCases son reutilizables en múltiples ViewModels.
- Los UseCases son más fáciles de testear en aislamiento.

---

## 4. Hilt — Inyección de Dependencias *(20 min)*

### 4.1 ¿Por qué inyección de dependencias?

```kotlin
// ❌ Sin DI: acoplamiento fuerte, imposible de testear
class ProductosViewModel : ViewModel() {
    private val repository = ProductosRepositoryImpl(
        ApiService.create(),        // ¿cómo reemplazar en tests?
        AppDatabase.getInstance()   // crea toda la DB real
    )
}

// ✅ Con Hilt: desacoplado, fácil de testear con mocks
@HiltViewModel
class ProductosViewModel @Inject constructor(
    private val getProductosUseCase: GetProductosUseCase  // Hilt lo inyecta
) : ViewModel()
```

### 4.2 Configuración de Hilt

```kotlin
// 1. Application class
@HiltAndroidApp
class MiApp : Application()

// 2. Activity (debe tener @AndroidEntryPoint)
@AndroidEntryPoint
class MainActivity : ComponentActivity() { ... }

// 3. Module para proveer dependencias
@Module
@InstallIn(SingletonComponent::class)
object AppModule {

    @Provides
    @Singleton
    fun provideRetrofit(): Retrofit = Retrofit.Builder()
        .baseUrl(BuildConfig.API_BASE_URL)
        .addConverterFactory(Json.asConverterFactory("application/json".toMediaType()))
        .build()

    @Provides
    @Singleton
    fun provideProductosApi(retrofit: Retrofit): ProductosApi =
        retrofit.create(ProductosApi::class.java)

    @Provides
    @Singleton
    fun provideProductosRepository(api: ProductosApi): ProductosRepository =
        ProductosRepositoryImpl(api)
}
```

---

## 5. Navegación Type-Safe con Jetpack Navigation 2.7+ *(20 min)*

### 5.1 El problema con la navegación antigua

```kotlin
// ❌ Navegación con strings — propenso a errores de typo, no type-safe
navController.navigate("producto_detalle/123")  // ¿y si el ID no es Int?
// No hay error en tiempo de compilación si escribes "producto_detallE"
```

### 5.2 Navegación type-safe con objetos Kotlin

```kotlin
// Definir rutas como objetos/data classes — verificadas en compilación
@Serializable
object RutaLista              // pantalla sin argumentos

@Serializable
data class RutaDetalle(val productoId: Int)  // pantalla con argumentos

@Serializable
object RutaCrear

// NavHost — el grafo de navegación
@Composable
fun AppNavHost(navController: NavHostController = rememberNavController()) {
    NavHost(
        navController = navController,
        startDestination = RutaLista
    ) {
        composable<RutaLista> {
            ListaScreen(
                onProductoClick = { id ->
                    navController.navigate(RutaDetalle(productoId = id))
                },
                onAgregarClick = {
                    navController.navigate(RutaCrear)
                }
            )
        }

        composable<RutaDetalle> { backStackEntry ->
            val ruta: RutaDetalle = backStackEntry.toRoute()
            DetalleScreen(
                productoId = ruta.productoId,
                onBack = navController::popBackStack
            )
        }

        composable<RutaCrear> {
            CrearScreen(onBack = navController::popBackStack)
        }
    }
}
```

### 5.3 Predictive Back Gesture (Android 13+)

El sistema permite **previsualizar** la pantalla anterior mientras el usuario desliza para atrás:

```kotlin
// En build.gradle.kts — habilitar predictive back
android {
    defaultConfig {
        // Requerido para predictive back gesture
    }
}

// En AndroidManifest.xml
// android:enableOnBackInvokedCallback="true"  en la etiqueta <application>
```

---

## 6. Resumen de la Sesión

| Concepto | Clave a recordar |
|---|---|
| Ciclo de vida | ViewModel sobrevive rotaciones; Activity no |
| MVVM + UDF | Eventos van UP (UI → VM), estado va DOWN (VM → UI). Nunca al revés. |
| StateFlow | Reemplaza LiveData. `collectAsStateWithLifecycle()` en Compose. |
| UseCase | 1 clase = 1 operación de negocio. Reutilizable, testeable. |
| Hilt | `@HiltAndroidApp`, `@AndroidEntryPoint`, `@HiltViewModel`, `@Inject`, `@Module` |
| Navigation | Rutas como objetos Kotlin `@Serializable`. Type-safe, verificado en compilación. |

---

## Tarea Previa al CodLab

1. Agrega las dependencias de Hilt a tu proyecto (en `build.gradle.kts`):
   - `implementation("com.google.dagger:hilt-android:2.51+")`
   - `kapt("com.google.dagger:hilt-compiler:2.51+")`
   - Plugin: `id("com.google.dagger.hilt.android")`
2. Crea la clase `Application` con `@HiltAndroidApp`.

---

## Recursos Adicionales

| Recurso | Enlace | Para qué |
|---|---|---|
| Guide to App Architecture | [developer.android.com/topic/architecture](https://developer.android.com/topic/architecture) | Guía oficial MVVM + Clean |
| Hilt Docs | [developer.android.com/training/dependency-injection/hilt-android](https://developer.android.com/training/dependency-injection/hilt-android) | Inyección de dependencias |
| Navigation Compose | [developer.android.com/guide/navigation/design](https://developer.android.com/guide/navigation) | Navegación type-safe |
| StateFlow vs LiveData | [developer.android.com/kotlin/flow/stateflow-and-sharedflow](https://developer.android.com/kotlin/flow/stateflow-and-sharedflow) | Comparativa detallada |
