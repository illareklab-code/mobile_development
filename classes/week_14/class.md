# Semana 14 — Testing, Debugging y Monitoreo de Rendimiento
**Curso:** Desarrollo de Sistemas Móviles | **Unidad:** U4  
**Sesión:** Presencial | **Duración:** 2 horas

---

## Objetivos de la Sesión

- Implementar unit tests para ViewModels con JUnit 5, MockK y Turbine.
- Escribir UI tests para composables con Compose Testing API.
- Usar Android Studio Profiler para detectar fugas de memoria y frames lentos.
- Configurar Firebase Crashlytics para monitoreo de crashes en producción.
- Aplicar property-based testing con Kotest.

---

## 1. La Pirámide de Testing en Android 2026 *(10 min)*

```
              ╔═══════════════╗
              ║   E2E Tests   ║  10% — más lentos, más caros
              ╠═══════════════╣
              ║    UI Tests   ║  20% — Compose Testing API
              ╠═══════════════╣
              ║  Unit Tests   ║  70% — más rápidos, aislados
              ╚═══════════════╝
```

**Distribución recomendada:**
- **70% Unit Tests:** ViewModel, UseCase, Repository, mapper functions — sin Android.
- **20% Integration Tests:** Room con in-memory DB, Retrofit con MockWebServer.
- **10% UI Tests:** flujos críticos end-to-end con Compose Testing API.

---

## 2. Unit Testing con JUnit 5 + MockK *(35 min)*

### 2.1 Configurar las dependencias

```kotlin
// app/build.gradle.kts
dependencies {
    // JUnit 5 (Jupiter)
    testImplementation("org.junit.jupiter:junit-jupiter:5.11.0")
    testImplementation("org.junit.jupiter:junit-jupiter-params:5.11.0")

    // MockK — mocking idiomático para Kotlin
    testImplementation("io.mockk:mockk:1.13.12")

    // Turbine — testing de Kotlin Flow
    testImplementation("app.cash.turbine:turbine:1.1.0")

    // Coroutines test
    testImplementation("org.jetbrains.kotlinx:kotlinx-coroutines-test:1.8.1")

    // Google Truth — assertions más legibles
    testImplementation("com.google.truth:truth:1.4.4")
}

// Necesario para JUnit 5
tasks.withType<Test> {
    useJUnitPlatform()
}
```

### 2.2 Testear un ViewModel con StateFlow

```kotlin
// ProductosViewModelTest.kt
class ProductosViewModelTest {

    // MockK crea un mock del repositorio
    private val repository = mockk<ProductosRepository>()
    private lateinit var viewModel: ProductosViewModel

    // TestCoroutineDispatcher: controla el tiempo en coroutines de test
    private val testDispatcher = StandardTestDispatcher()

    @BeforeEach
    fun setup() {
        Dispatchers.setMain(testDispatcher)
    }

    @AfterEach
    fun teardown() {
        Dispatchers.resetMain()
    }

    @Test
    fun `cargar productos exitosamente actualiza estado a Success`() = runTest {
        // GIVEN
        val productosEsperados = listOf(
            Producto(1, "Laptop", 5200.0, "Tech"),
            Producto(2, "Auriculares", 890.0, "Tech")
        )
        coEvery { repository.getProductos() } returns productosEsperados

        viewModel = ProductosViewModel(GetProductosUseCase(repository))

        // WHEN + THEN — Turbine para testear Flow
        viewModel.uiState.test {
            val loading = awaitItem()
            assertThat(loading.isLoading).isTrue()

            testDispatcher.scheduler.advanceUntilIdle()  // ejecutar todas las coroutines pendientes

            val success = awaitItem()
            assertThat(success.isLoading).isFalse()
            assertThat(success.productos).isEqualTo(productosEsperados)
            assertThat(success.error).isNull()
        }
    }

    @Test
    fun `error de red actualiza estado con mensaje de error`() = runTest {
        // GIVEN
        coEvery { repository.getProductos() } throws IOException("Sin conexión")

        viewModel = ProductosViewModel(GetProductosUseCase(repository))

        // WHEN + THEN
        viewModel.uiState.test {
            skipItems(1)  // skip loading state
            testDispatcher.scheduler.advanceUntilIdle()

            val errorState = awaitItem()
            assertThat(errorState.isLoading).isFalse()
            assertThat(errorState.error).isNotNull()
            assertThat(errorState.productos).isEmpty()
        }
    }

    @Test
    fun `estado inicial es Loading`() = runTest {
        coEvery { repository.getProductos() } returns emptyList()
        viewModel = ProductosViewModel(GetProductosUseCase(repository))

        val estadoInicial = viewModel.uiState.value
        assertThat(estadoInicial.isLoading).isTrue()
    }

    @ParameterizedTest
    @ValueSource(strings = ["Laptop", "LAPTOP", "laptop", "lapt"])
    fun `busqueda es case-insensitive`(query: String) = runTest {
        val productos = listOf(Producto(1, "Laptop Dell", 5200.0, "Tech"))
        coEvery { repository.buscar(query) } returns productos.filter {
            it.nombre.contains(query, ignoreCase = true)
        }
        viewModel = ProductosViewModel(GetProductosUseCase(repository))
        viewModel.onBusquedaChange(query)
        testDispatcher.scheduler.advanceUntilIdle()

        val state = viewModel.uiState.value
        assertThat(state.productos).isNotEmpty()
    }
}
```

### 2.3 MockK — Mock de dependencias

```kotlin
// Crear mocks
val repository = mockk<ProductosRepository>()
val mockedList = mockk<List<Producto>>(relaxed = true)  // relaxed: retorna defaults para métodos no configurados

// Configurar comportamiento
coEvery { repository.getProductos() } returns listOf(producto1, producto2)  // suspend function
every { repository.getProductosSync() } returns listOf(producto1)           // función regular
coEvery { repository.guardar(any()) } just Runs                             // suspend void

// Verificar que fue llamado
coVerify(exactly = 1) { repository.getProductos() }
coVerify(atLeast = 1) { repository.guardar(producto1) }

// Capturar argumentos
val slot = slot<Producto>()
coEvery { repository.guardar(capture(slot)) } just Runs
// Después: assertThat(slot.captured.nombre).isEqualTo("Laptop")
```

---

## 3. Compose Testing API *(25 min)*

### 3.1 Configuración

```kotlin
// app/build.gradle.kts
androidTestImplementation("androidx.compose.ui:ui-test-junit4")
debugImplementation("androidx.compose.ui:ui-test-manifest")
```

### 3.2 Tests de UI con Compose

```kotlin
// ProductosScreenTest.kt
@RunWith(AndroidJUnit4::class)
class ProductosScreenTest {

    @get:Rule
    val composeTestRule = createComposeRule()

    @Test
    fun `lista de productos muestra los items correctamente`() {
        val productos = listOf(
            Producto(1, "Laptop Dell XPS", 5200.0, "Tech"),
            Producto(2, "Auriculares Sony", 890.0, "Tech")
        )

        composeTestRule.setContent {
            MiAppTheme {
                ProductosList(productos = productos, onProductoClick = {})
            }
        }

        // Verificar que los textos están en pantalla
        composeTestRule.onNodeWithText("Laptop Dell XPS").assertIsDisplayed()
        composeTestRule.onNodeWithText("Auriculares Sony").assertIsDisplayed()
    }

    @Test
    fun `click en producto navega al detalle`() {
        var productoClickeado: Int? = null
        val productos = listOf(Producto(1, "Laptop", 5200.0, "Tech"))

        composeTestRule.setContent {
            ProductosList(
                productos = productos,
                onProductoClick = { id -> productoClickeado = id }
            )
        }

        composeTestRule.onNodeWithText("Laptop").performClick()
        assertThat(productoClickeado).isEqualTo(1)
    }

    @Test
    fun `loading indicator visible cuando isLoading es true`() {
        composeTestRule.setContent {
            ProductosScreen(uiState = ProductosUiState(isLoading = true))
        }

        composeTestRule
            .onNodeWithContentDescription("Cargando")  // requiere contentDescription
            .assertIsDisplayed()
    }
}
```

### 3.3 Semantic Tags para testabilidad

```kotlin
// En el código de producción — agrega testTags para facilitar el testing
@Composable
fun BotonConfirmar(onClick: () -> Unit) {
    Button(
        onClick = onClick,
        modifier = Modifier.semantics { testTag = "btn_confirmar" }
    ) {
        Text("Confirmar")
    }
}

// En el test:
composeTestRule.onNodeWithTag("btn_confirmar").performClick()
```

---

## 4. Property-Based Testing con Kotest *(15 min)*

En lugar de pensar en casos de prueba individuales, Kotest genera miles de inputs aleatorios:

```kotlin
testImplementation("io.kotest:kotest-runner-junit5:5.9.1")
testImplementation("io.kotest:kotest-property:5.9.1")

class PrecioProductoTest : FunSpec({

    test("el precio siempre debe ser positivo después de aplicar descuento") {
        forAll(
            Arb.double(0.01, 10000.0),   // precio base aleatorio
            Arb.double(0.0, 0.5)         // descuento aleatorio entre 0% y 50%
        ) { precio, descuento ->
            val precioConDescuento = aplicarDescuento(precio, descuento)
            precioConDescuento > 0  // propiedad que siempre debe cumplirse
        }
    }

    test("serialización y deserialización son inversas") {
        forAll(
            Arb.string(1..100),    // nombre aleatorio
            Arb.double(0.01, 10000.0)  // precio aleatorio
        ) { nombre, precio ->
            val producto = Producto(1, nombre, precio, "Test")
            val json = Json.encodeToString(producto)
            val deserializado = Json.decodeFromString<Producto>(json)
            deserializado == producto
        }
    }
})
```

---

## 5. Android Studio Profiler *(15 min)*

Abre el Profiler con `View → Tool Windows → App Inspection` o directamente `Profiler`.

### 5.1 Memory Profiler — Detectar fugas de memoria

**Síntoma de fuga:** el gráfico de memoria sube gradualmente y nunca baja aunque navegas entre pantallas.

```kotlin
// Causa común: listeners/callbacks que retienen referencias a la Activity
class MiFragment : Fragment() {
    // ❌ MAL: listener anónimo retiene referencia al Fragment
    val listener = View.OnClickListener { /* usa 'this' */ }

    // ✅ BIEN: desregistrar en onDestroyView
    override fun onDestroyView() {
        super.onDestroyView()
        binding = null  // liberar el binding
    }
}

// En Compose esto es menos común porque los composables no tienen ciclo de vida propio
// Pero sí puede ocurrir con callbacks que capturan el contexto
```

**Cómo detectar:**
1. Profiler → Memory.
2. Navegar varias veces a la pantalla sospechosa y volver.
3. Forzar GC (botón de basura en el profiler).
4. Si el memory sigue creciendo → fuga.
5. Tomar un heap dump y buscar instancias de la pantalla.

### 5.2 CPU Profiler — Detectar frames lentos (jank)

Cualquier frame que tarde más de **16ms** (60fps) o **8ms** (120fps) causa "jank" (vibración visual):

1. CPU Profiler → "System Trace".
2. Interactúa con la app.
3. Busca frames en rojo (> 16ms) en el timeline.
4. Identifica qué código corre en el Main Thread durante esos frames.

**Regla:** **todo trabajo pesado** (red, BD, parsing) debe ser en `Dispatchers.IO`, **nunca** en el Main Thread.

---

## 6. Firebase Crashlytics *(10 min)*

```kotlin
implementation(platform("com.google.firebase:firebase-bom:33.0.0"))
implementation("com.google.firebase:firebase-crashlytics")
implementation("com.google.firebase:firebase-analytics")  // requerido para Crashlytics

// Reportar excepción no-fatal (no cierra la app pero la registra en Crashlytics)
try {
    operacionRiesgosa()
} catch (e: Exception) {
    Firebase.crashlytics.recordException(e)
}

// Agregar contexto para debugging
Firebase.crashlytics.setCustomKey("user_id", userId)
Firebase.crashlytics.setCustomKey("screen", "ProductosList")
Firebase.crashlytics.log("Usuario intentó eliminar producto $productoId")

// Habilitar/deshabilitar según el consentimiento del usuario
Firebase.crashlytics.setCrashlyticsCollectionEnabled(!BuildConfig.DEBUG)
```

---

## 7. Resumen de la Sesión

| Concepto | Clave a recordar |
|---|---|
| Pirámide de testing | 70% unit, 20% integration, 10% UI. Los unit tests son los más valiosos por frecuencia. |
| JUnit 5 | `@Test`, `@ParameterizedTest`, `@BeforeEach`, `@AfterEach`. `useJUnitPlatform()` en Gradle. |
| MockK | `mockk<Tipo>()`, `coEvery { } returns`, `coVerify { }`. `slot<T>()` para capturar argumentos. |
| Turbine | `flow.test { awaitItem() }` para testear StateFlow y Flow. |
| Compose Testing | `composeTestRule.onNodeWithText().performClick()`. Usa `testTag` para elementos sin texto. |
| Profiler | Memory: buscar crecimientos sin GC. CPU: frames > 16ms en Main Thread. |
| Crashlytics | `recordException()` para no-fatales. `setCustomKey()` para contexto de debug. |

---

## Recursos Adicionales

| Recurso | Enlace | Para qué |
|---|---|---|
| JUnit 5 Docs | [junit.org/junit5](https://junit.org/junit5/) | Referencia de anotaciones |
| MockK Docs | [mockk.io](https://mockk.io/) | Guía completa de mocking |
| Compose Testing | [developer.android.com/jetpack/compose/testing](https://developer.android.com/jetpack/compose/testing) | API de testing de composables |
| Turbine | [github.com/cashapp/turbine](https://github.com/cashapp/turbine) | Testing de Kotlin Flows |
| Firebase Crashlytics | [firebase.google.com/docs/crashlytics](https://firebase.google.com/docs/crashlytics) | Setup y uso |
| ANR causes | [developer.android.com/topic/performance/vitals/anr](https://developer.android.com/topic/performance/vitals/anr) | Causas de ANR en Android |
