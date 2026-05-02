# CodLab N°3 — Material Design 3 con Jetpack Compose
**Curso:** Desarrollo de Sistemas Móviles | **Semana:** 3  
**Evaluación:** Individual

---

## Objetivos

- Aplicar un tema Material Design 3 con colores dinámicos.
- Implementar los componentes principales: TopAppBar, Card, LazyColumn, TextField.
- Crear un toggle de tema claro/oscuro funcional.
- Usar data classes y colecciones de Kotlin en la UI.
- Cumplir mínimos de accesibilidad (contentDescription, tamaño de toque).

## Prerrequisitos

- CodLab N°2 completado.
- Cuenta en [Figma](https://figma.com) creada (para usar Material Theme Builder).

## Herramientas Requeridas

- Android Studio Meerkat
- Material Theme Builder: [m3.material.io/theme-builder](https://m3.material.io/theme-builder)

---

## Paso 1: Crear proyecto y configurar tema

Crea un proyecto nuevo: `MaterialDesign3Lab`. Luego:

1. Abre [m3.material.io/theme-builder](https://m3.material.io/theme-builder).
2. Elige un color primario de tu preferencia (ej: un morado o verde).
3. Clic en **Export** → **Jetpack Compose (Material3)**.
4. Reemplaza los archivos `Color.kt` y `Theme.kt` generados en `ui/theme/`.

Modifica `Theme.kt` para soportar toggle de tema:

```kotlin
// Agrega un CompositionLocal para controlar el tema desde cualquier lugar
val LocalDarkTheme = compositionLocalOf { false }

@Composable
fun MaterialDesign3LabTheme(
    darkTheme: Boolean = isSystemInDarkTheme(),
    dynamicColor: Boolean = true,
    content: @Composable () -> Unit
) {
    val colorScheme = when {
        dynamicColor && Build.VERSION.SDK_INT >= Build.VERSION_CODES.S -> {
            if (darkTheme) dynamicDarkColorScheme(LocalContext.current)
            else dynamicLightColorScheme(LocalContext.current)
        }
        darkTheme -> DarkColorScheme
        else -> LightColorScheme
    }

    CompositionLocalProvider(LocalDarkTheme provides darkTheme) {
        MaterialTheme(
            colorScheme = colorScheme,
            typography = Typography,
            content = content
        )
    }
}
```

---

## Paso 2: Modelo de datos

Crea `data/Producto.kt`:

```kotlin
data class Producto(
    val id: Int,
    val nombre: String,
    val descripcion: String,
    val precio: Double,
    val categoria: Categoria,
    val disponible: Boolean = true
)

enum class Categoria(val etiqueta: String, val icono: ImageVector) {
    TECNOLOGIA("Tecnología", Icons.Filled.Laptop),
    ROPA("Ropa", Icons.Filled.Checkroom),
    ALIMENTOS("Alimentos", Icons.Filled.LocalGroceryStore),
    HOGAR("Hogar", Icons.Filled.Home)
}

// Datos de prueba
object ProductosData {
    val lista = listOf(
        Producto(1, "Laptop Dell XPS 15", "Intel i9, 32GB RAM, RTX 4070", 5200.0, Categoria.TECNOLOGIA),
        Producto(2, "Auriculares Sony WH-1000XM5", "Cancelación de ruido activa", 890.0, Categoria.TECNOLOGIA),
        Producto(3, "Zapatillas Nike Air Max", "Talla 42, Color negro", 450.0, Categoria.ROPA),
        Producto(4, "Cafetera Nespresso", "Cápsulas compatibles, 19 bar", 320.0, Categoria.HOGAR),
        Producto(5, "Polo de algodón pima", "100% algodón peruano, talla M", 85.0, Categoria.ROPA),
        Producto(6, "Teclado mecánico Keychron K2", "Switches Brown, Bluetooth", 380.0, Categoria.TECNOLOGIA, disponible = false)
    )
}
```

---

## Paso 3: Pantalla principal con Scaffold + TopAppBar + lista

```kotlin
@Composable
fun ProductosScreen(modifier: Modifier = Modifier) {
    var oscuro by remember { mutableStateOf(false) }
    var busqueda by remember { mutableStateOf("") }
    var categoriaFiltro by remember { mutableStateOf<Categoria?>(null) }

    MaterialDesign3LabTheme(darkTheme = oscuro) {
        Surface(modifier = modifier.fillMaxSize()) {
            ProductosScaffold(
                oscuro = oscuro,
                busqueda = busqueda,
                categoriaFiltro = categoriaFiltro,
                onToggleTema = { oscuro = !oscuro },
                onBusquedaChange = { busqueda = it },
                onCategoriaChange = { categoriaFiltro = it }
            )
        }
    }
}

@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun ProductosScaffold(
    oscuro: Boolean,
    busqueda: String,
    categoriaFiltro: Categoria?,
    onToggleTema: () -> Unit,
    onBusquedaChange: (String) -> Unit,
    onCategoriaChange: (Categoria?) -> Unit
) {
    val productosFiltrados = ProductosData.lista
        .filter { busqueda.isEmpty() || it.nombre.contains(busqueda, ignoreCase = true) }
        .filter { categoriaFiltro == null || it.categoria == categoriaFiltro }

    Scaffold(
        topBar = {
            TopAppBar(
                title = { Text("Catálogo") },
                actions = {
                    // Toggle tema oscuro/claro
                    IconButton(onClick = onToggleTema) {
                        Icon(
                            imageVector = if (oscuro) Icons.Filled.LightMode else Icons.Filled.DarkMode,
                            contentDescription = if (oscuro) "Cambiar a tema claro" else "Cambiar a tema oscuro"
                        )
                    }
                }
            )
        },
        floatingActionButton = {
            FloatingActionButton(onClick = { /* agregar producto */ }) {
                Icon(Icons.Filled.Add, contentDescription = "Agregar producto")
            }
        }
    ) { padding ->
        Column(modifier = Modifier.padding(padding)) {
            // Barra de búsqueda
            OutlinedTextField(
                value = busqueda,
                onValueChange = onBusquedaChange,
                modifier = Modifier
                    .fillMaxWidth()
                    .padding(horizontal = 16.dp, vertical = 8.dp),
                placeholder = { Text("Buscar productos...") },
                leadingIcon = { Icon(Icons.Filled.Search, contentDescription = null) },
                trailingIcon = {
                    if (busqueda.isNotEmpty()) {
                        IconButton(onClick = { onBusquedaChange("") }) {
                            Icon(Icons.Filled.Clear, contentDescription = "Limpiar búsqueda")
                        }
                    }
                },
                singleLine = true
            )

            // Chips de categoría
            LazyRow(
                contentPadding = PaddingValues(horizontal = 16.dp),
                horizontalArrangement = Arrangement.spacedBy(8.dp)
            ) {
                item {
                    FilterChip(
                        selected = categoriaFiltro == null,
                        onClick = { onCategoriaChange(null) },
                        label = { Text("Todos") }
                    )
                }
                items(Categoria.entries) { cat ->
                    FilterChip(
                        selected = categoriaFiltro == cat,
                        onClick = { onCategoriaChange(if (categoriaFiltro == cat) null else cat) },
                        label = { Text(cat.etiqueta) },
                        leadingIcon = { Icon(cat.icono, contentDescription = null, Modifier.size(18.dp)) }
                    )
                }
            }

            // Lista de productos
            if (productosFiltrados.isEmpty()) {
                Box(modifier = Modifier.fillMaxSize(), contentAlignment = Alignment.Center) {
                    Text("No se encontraron productos", style = MaterialTheme.typography.bodyLarge)
                }
            } else {
                LazyColumn(
                    contentPadding = PaddingValues(16.dp),
                    verticalArrangement = Arrangement.spacedBy(8.dp)
                ) {
                    items(productosFiltrados, key = { it.id }) { producto ->
                        ProductoCard(producto = producto)
                    }
                }
            }
        }
    }
}

@Composable
fun ProductoCard(producto: Producto) {
    Card(
        modifier = Modifier.fillMaxWidth(),
        colors = CardDefaults.cardColors(
            containerColor = if (!producto.disponible)
                MaterialTheme.colorScheme.surfaceVariant
            else
                MaterialTheme.colorScheme.surface
        )
    ) {
        Row(
            modifier = Modifier.padding(16.dp),
            verticalAlignment = Alignment.CenterVertically,
            horizontalArrangement = Arrangement.spacedBy(12.dp)
        ) {
            // Ícono de categoría
            Surface(
                shape = MaterialTheme.shapes.medium,
                color = MaterialTheme.colorScheme.secondaryContainer
            ) {
                Icon(
                    imageVector = producto.categoria.icono,
                    contentDescription = null,
                    modifier = Modifier.padding(12.dp),
                    tint = MaterialTheme.colorScheme.onSecondaryContainer
                )
            }

            Column(modifier = Modifier.weight(1f)) {
                Text(producto.nombre, style = MaterialTheme.typography.titleMedium)
                Text(
                    text = producto.descripcion,
                    style = MaterialTheme.typography.bodySmall,
                    color = MaterialTheme.colorScheme.onSurfaceVariant,
                    maxLines = 1
                )
                Row(
                    horizontalArrangement = Arrangement.spacedBy(8.dp),
                    verticalAlignment = Alignment.CenterVertically
                ) {
                    AssistChip(
                        onClick = {},
                        label = { Text(producto.categoria.etiqueta) }
                    )
                    if (!producto.disponible) {
                        Badge { Text("Agotado") }
                    }
                }
            }

            Text(
                text = "S/ ${"%.2f".format(producto.precio)}",
                style = MaterialTheme.typography.titleMedium,
                color = MaterialTheme.colorScheme.primary
            )
        }
    }
}
```

---

## Paso 4: Estado vacío y Snackbar

```kotlin
// Agrega un SnackbarHostState al Scaffold para confirmaciones
val snackbarHostState = remember { SnackbarHostState() }
val scope = rememberCoroutineScope()

// En el FAB:
FloatingActionButton(onClick = {
    scope.launch {
        snackbarHostState.showSnackbar(
            message = "Función en desarrollo",
            actionLabel = "OK"
        )
    }
}) { Icon(Icons.Filled.Add, contentDescription = "Agregar") }

// Agrega al Scaffold: snackbarHost = { SnackbarHost(snackbarHostState) }
```

---

## Entregables

1. App corriendo con lista de productos mostrando Cards con información.
2. Toggle de tema claro/oscuro funcional desde el TopAppBar.
3. Filtro por categoría (FilterChips) funcionando.
4. Barra de búsqueda funcional (filtra por nombre en tiempo real).
5. Badge "Agotado" visible en el producto no disponible.

## Criterios de Evaluación

| Criterio | Puntaje |
|---|---|
| Tema M3 correctamente configurado (colores dinámicos) | 2 pts |
| Lista de productos con Cards y datos correctos | 2 pts |
| Toggle tema oscuro/claro funcional | 2 pts |
| Filtro por categoría con FilterChips | 2 pts |
| contentDescription en todos los iconos interactivos | 1 pt |
| Búsqueda en tiempo real | 1 pt |
| **Total** | **10 pts** |
