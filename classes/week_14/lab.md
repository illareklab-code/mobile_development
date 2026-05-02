# CodLab N°12 — Unit Tests + Compose UI Tests + Profiler
**Curso:** Desarrollo de Sistemas Móviles | **Semana:** 14  
**Evaluación:** Individual

---

## Objetivos

- Escribir mínimo 5 unit tests para el ViewModel del proyecto del curso.
- Implementar 2 UI tests para pantallas con Compose Testing API.
- Usar semantic tags para hacer los composables testeable.
- Detectar y corregir un problema de rendimiento con el Profiler.
- Integrar Firebase Crashlytics en el proyecto.

## Prerrequisitos

- CodLab N°11 completado.
- Proyecto del curso con ViewModel y StateFlow funcionando.

---

## Paso 1: Agregar dependencias de testing

```kotlin
// app/build.gradle.kts
dependencies {
    // JUnit 5
    testImplementation("org.junit.jupiter:junit-jupiter:5.11.0")
    testImplementation("org.junit.jupiter:junit-jupiter-params:5.11.0")

    // MockK
    testImplementation("io.mockk:mockk:1.13.12")

    // Turbine (testing de Flow/StateFlow)
    testImplementation("app.cash.turbine:turbine:1.1.0")

    // Coroutines Test
    testImplementation("org.jetbrains.kotlinx:kotlinx-coroutines-test:1.8.1")

    // Google Truth
    testImplementation("com.google.truth:truth:1.4.4")

    // Compose UI Testing
    androidTestImplementation("androidx.compose.ui:ui-test-junit4")
    debugImplementation("androidx.compose.ui:ui-test-manifest")
}

tasks.withType<Test> {
    useJUnitPlatform()
}
```

---

## Paso 2: Unit tests del ViewModel del proyecto

Crea `src/test/kotlin/.../[TuViewModel]Test.kt`:

```kotlin
class [TuViewModel]Test {

    private val repository = mockk<[TuRepository]>()
    private lateinit var viewModel: [TuViewModel]
    private val testDispatcher = StandardTestDispatcher()

    @BeforeEach
    fun setup() {
        Dispatchers.setMain(testDispatcher)
    }

    @AfterEach
    fun teardown() {
        Dispatchers.resetMain()
    }

    // TEST 1: Estado inicial
    @Test
    fun `estado inicial debe ser Loading`() = runTest {
        coEvery { repository.getData() } returns emptyList()
        viewModel = [TuViewModel](repository)

        assertThat(viewModel.uiState.value.isLoading).isTrue()
    }

    // TEST 2: Carga exitosa
    @Test
    fun `carga exitosa actualiza lista y quita loading`() = runTest {
        val datosEsperados = listOf(/* tus objetos de prueba */)
        coEvery { repository.getData() } returns datosEsperados

        viewModel = [TuViewModel](repository)

        viewModel.uiState.test {
            val loading = awaitItem()
            assertThat(loading.isLoading).isTrue()

            testDispatcher.scheduler.advanceUntilIdle()

            val success = awaitItem()
            assertThat(success.isLoading).isFalse()
            assertThat(success.items).isEqualTo(datosEsperados)
        }
    }

    // TEST 3: Error de red
    @Test
    fun `error de red actualiza estado con mensaje de error`() = runTest {
        coEvery { repository.getData() } throws Exception("Error de red")

        viewModel = [TuViewModel](repository)

        viewModel.uiState.test {
            skipItems(1)  // skip loading
            testDispatcher.scheduler.advanceUntilIdle()

            val errorState = awaitItem()
            assertThat(errorState.error).isNotNull()
            assertThat(errorState.isLoading).isFalse()
        }
    }

    // TEST 4: Verificar que el repositorio fue llamado
    @Test
    fun `al iniciar el ViewModel se llama al repositorio exactamente una vez`() = runTest {
        coEvery { repository.getData() } returns emptyList()
        viewModel = [TuViewModel](repository)
        testDispatcher.scheduler.advanceUntilIdle()

        coVerify(exactly = 1) { repository.getData() }
    }

    // TEST 5: Parámetros múltiples con @ParameterizedTest
    @ParameterizedTest
    @ValueSource(ints = [1, 5, 10, 100])
    fun `eliminar item reduce el tamaño de la lista`(cantidadInicial: Int) = runTest {
        val items = (1..cantidadInicial).map { crearItemDePrueba(it) }
        coEvery { repository.getData() } returns items
        coEvery { repository.delete(any()) } just Runs

        viewModel = [TuViewModel](repository)
        testDispatcher.scheduler.advanceUntilIdle()

        viewModel.eliminar(items.first().id)
        testDispatcher.scheduler.advanceUntilIdle()

        coVerify { repository.delete(items.first().id) }
    }
}
```

> Reemplaza `[TuViewModel]`, `[TuRepository]` y los métodos con los nombres reales de tu proyecto.

---

## Paso 3: Agregar semantic tags para testing en producción

Busca los elementos más importantes de tu UI y agrega `testTag`:

```kotlin
// En los composables del proyecto
@Composable
fun ListaItems(items: List<Item>, onItemClick: (Int) -> Unit) {
    LazyColumn(
        modifier = Modifier.semantics { testTag = "lista_items" }
    ) {
        items(items, key = { it.id }) { item ->
            Card(
                onClick = { onItemClick(item.id) },
                modifier = Modifier.semantics { testTag = "item_${item.id}" }
            ) { ... }
        }
    }
}

@Composable
fun BotonAgregar(onClick: () -> Unit) {
    FloatingActionButton(
        onClick = onClick,
        modifier = Modifier.semantics { testTag = "fab_agregar" }
    ) {
        Icon(Icons.Filled.Add, contentDescription = "Agregar")
    }
}

@Composable
fun LoadingIndicator() {
    CircularProgressIndicator(
        modifier = Modifier.semantics { contentDescription = "Cargando contenido" }
    )
}
```

---

## Paso 4: UI Tests con Compose Testing API

```kotlin
// src/androidTest/kotlin/.../[TuPantalla]Test.kt
@RunWith(AndroidJUnit4::class)
class [TuPantalla]Test {

    @get:Rule
    val composeTestRule = createComposeRule()

    // UI TEST 1: Lista visible con datos
    @Test
    fun `lista muestra los items cuando hay datos`() {
        val testItems = listOf(
            // tus objetos de prueba
        )

        composeTestRule.setContent {
            MiAppTheme {
                [TuListaScreen](
                    items = testItems,
                    isLoading = false,
                    onItemClick = {}
                )
            }
        }

        // Verificar que el primer item aparece en pantalla
        composeTestRule.onNodeWithText(testItems.first().titulo).assertIsDisplayed()
        composeTestRule.onNodeWithTag("lista_items").assertIsDisplayed()
    }

    // UI TEST 2: Loading indicator visible
    @Test
    fun `loading indicator visible cuando isLoading es true`() {
        composeTestRule.setContent {
            MiAppTheme {
                [TuListaScreen](
                    items = emptyList(),
                    isLoading = true,
                    onItemClick = {}
                )
            }
        }

        composeTestRule
            .onNodeWithContentDescription("Cargando contenido")
            .assertIsDisplayed()
    }

    // UI TEST 3: Click en FAB invoca el callback
    @Test
    fun `click en FAB invoca el callback de agregar`() {
        var fueClickeado = false

        composeTestRule.setContent {
            MiAppTheme {
                [TuPantalla](
                    onAgregarClick = { fueClickeado = true }
                )
            }
        }

        composeTestRule.onNodeWithTag("fab_agregar").performClick()
        assertThat(fueClickeado).isTrue()
    }
}
```

---

## Paso 5: Firebase Crashlytics

```kotlin
// app/build.gradle.kts
implementation("com.google.firebase:firebase-crashlytics")

// app/build.gradle.kts — plugin
plugins {
    id("com.google.firebase.crashlytics")
}
```

```kotlin
// En el Application class o en puntos críticos del código:

// Desactivar en debug (evitar ruido en Crashlytics durante desarrollo)
Firebase.crashlytics.setCrashlyticsCollectionEnabled(!BuildConfig.DEBUG)

// Agregar context útil para debugging
class ProductosViewModel @Inject constructor(...) : ViewModel() {
    init {
        Firebase.crashlytics.setCustomKey("viewmodel", "ProductosViewModel")
    }

    fun cargar() {
        viewModelScope.launch {
            try {
                val datos = repository.getData()
                Firebase.crashlytics.log("Productos cargados: ${datos.size}")
                _uiState.update { it.copy(items = datos) }
            } catch (e: Exception) {
                Firebase.crashlytics.recordException(e)  // log sin crashear la app
                _uiState.update { it.copy(error = e.message) }
            }
        }
    }
}
```

---

## Paso 6: Usar el Profiler para detectar problemas

1. Ejecuta la app en el emulador.
2. Abre **Profiler** en Android Studio (View → Tool Windows → Profiler).
3. Navega rápidamente entre pantallas 5+ veces.
4. Observa el gráfico de **Memoria**:
   - ¿Sube constantemente? → posible fuga.
   - ¿Baja después del GC? → normal.
5. En el **CPU Profiler**, graba mientras haces scroll en una lista larga.
6. Documenta lo que encontraste (captura de pantalla del profiler).

Si encuentras un problema de rendimiento:
- Muévelo a `Dispatchers.IO` si es trabajo de I/O.
- Usa `key = { it.id }` en `LazyColumn` si el problema es el reordenamiento.
- Verifica que no estás creando objetos dentro de composables sin `remember`.

---

## Entregables

1. Archivo de tests con mínimo 5 unit tests que pasan (`./gradlew test`).
2. Archivo de UI tests con mínimo 2 tests que pasan (`./gradlew connectedAndroidTest`).
3. Capturas de pantalla del resultado de los tests en Android Studio.
4. Captura del Profiler mostrando el análisis de memoria/CPU.
5. Firebase Crashlytics integrado (visible en Firebase Console al forzar un error de prueba).

## Criterios de Evaluación

| Criterio | Puntaje |
|---|---|
| 5 unit tests pasan correctamente (mock del repositorio) | 3 pts |
| 2 UI tests pasan con Compose Testing API | 2 pts |
| Semantic tags en al menos 3 elementos de UI en producción | 1 pt |
| Captura del Profiler con análisis documentado | 2 pts |
| Crashlytics integrado y `recordException` usado en al menos 1 lugar | 2 pts |
| **Total** | **10 pts** |
