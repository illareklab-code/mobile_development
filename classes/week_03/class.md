# Semana 3 — Material Design 3 y UI con Jetpack Compose
**Curso:** Desarrollo de Sistemas Móviles | **Unidad:** U1  
**Sesión:** Presencial | **Duración:** 2 horas

---

## Objetivos de la Sesión

- Comprender la filosofía y sistema de diseño de Material Design 3 (Material You).
- Aplicar el sistema de color dinámico y roles de color de M3.
- Implementar componentes de UI de Material Design 3 con Jetpack Compose.
- Conocer los estándares de accesibilidad WCAG 2.2 para aplicaciones móviles.
- Optimizar el rendimiento de Compose con Baseline Profiles.

---

## 1. Material Design 3 — La Evolución del Diseño en Android *(20 min)*

### 1.1 De Material Design 1 a Material You

| Versión | Año | Característica principal |
|---|---|---|
| Material Design 1.0 | 2014 | Diseño plano con sombras y capas. Paleta fija (azul primario, etc.) |
| Material Design 2.0 | 2018 | Más espacio en blanco, tipografía Roboto, formas redondeadas |
| **Material Design 3 (Material You)** | **2021–2026** | **Colores dinámicos basados en el wallpaper del usuario** |

> **Importante 2026:** Google ya no actualiza Material Design 2. Toda nueva app DEBE usar Material Design 3.

### 1.2 La filosofía de Material You

**Material You** introduce la personalización como principio central:
- Los colores de la app se adaptan automáticamente al **wallpaper** del usuario (Android 12+).
- El sistema genera una **paleta completa de 5 colores clave** a partir del wallpaper usando el algoritmo Monet.
- En Android < 12: el sistema usa una paleta predeterminada (sin personalización).

### 1.3 El sistema de color de Material Design 3

M3 define **roles de color** (no colores fijos):

```
Primary      → color principal de la app (botones, FAB)
  on-Primary → texto/iconos SOBRE el color primary

Secondary    → color de apoyo (chips, toggle buttons)
  on-Secondary

Tertiary     → color de acento adicional
  on-Tertiary

Error        → alertas, campos inválidos
  on-Error

Surface      → fondo de cards, sheets, dialogs
  on-Surface → texto sobre superficies

Background   → fondo de la pantalla principal
  on-Background

Outline      → bordes de TextFields, separadores
```

**¿Por qué roles en lugar de colores fijos?**
Porque el sistema puede intercambiar el tema claro/oscuro automáticamente sin que el desarrollador cambie el código. Solo cambian los valores de los roles, no su uso.

### 1.4 Implementación del tema en Compose

```kotlin
// ui/theme/Color.kt — define colores base
val md_theme_light_primary = Color(0xFF6750A4)
val md_theme_dark_primary = Color(0xFFD0BCFF)
// ...

// ui/theme/Theme.kt
@Composable
fun MiAppTheme(
    darkTheme: Boolean = isSystemInDarkTheme(),
    dynamicColor: Boolean = true,  // colores dinámicos del wallpaper
    content: @Composable () -> Unit
) {
    val colorScheme = when {
        dynamicColor && Build.VERSION.SDK_INT >= Build.VERSION_CODES.S -> {
            val context = LocalContext.current
            if (darkTheme) dynamicDarkColorScheme(context)  // colores del wallpaper en dark
            else dynamicLightColorScheme(context)           // colores del wallpaper en light
        }
        darkTheme -> DarkColorScheme
        else -> LightColorScheme
    }
    MaterialTheme(
        colorScheme = colorScheme,
        typography = Typography,
        content = content
    )
}
```

> **Herramienta:** genera tu paleta M3 personalizada en [m3.material.io/theme-builder](https://m3.material.io/theme-builder).

---

## 2. Componentes de Material Design 3 en Compose *(35 min)*

### 2.1 Botones

M3 tiene 5 tipos de botones según su nivel de énfasis:

```kotlin
// Mayor énfasis → menor énfasis

Button(onClick = { })              { Text("Filled") }          // acción principal
ElevatedButton(onClick = { })      { Text("Elevated") }        // acción alternativa
FilledTonalButton(onClick = { })   { Text("Tonal") }           // énfasis medio
OutlinedButton(onClick = { })      { Text("Outlined") }        // énfasis secundario
TextButton(onClick = { })          { Text("Text") }            // menor énfasis
```

**Regla:** en una pantalla, usa máximo 1 botón `Filled` (la acción primaria). El resto son de menor jerarquía.

### 2.2 Campos de texto

```kotlin
var texto by remember { mutableStateOf("") }
var error by remember { mutableStateOf(false) }

OutlinedTextField(
    value = texto,
    onValueChange = { 
        texto = it
        error = it.length > 50
    },
    label = { Text("Nombre del producto") },
    placeholder = { Text("Ej: Laptop Dell XPS") },
    supportingText = { 
        if (error) Text("Máximo 50 caracteres", color = MaterialTheme.colorScheme.error)
        else Text("${texto.length}/50")
    },
    isError = error,
    trailingIcon = {
        if (error) Icon(Icons.Filled.Error, contentDescription = "Error")
    },
    singleLine = true,
    modifier = Modifier.fillMaxWidth()
)
```

### 2.3 Cards

```kotlin
// Card básico
Card(
    modifier = Modifier.fillMaxWidth(),
    elevation = CardDefaults.cardElevation(defaultElevation = 2.dp)
) {
    Column(modifier = Modifier.padding(16.dp)) {
        Text("Título", style = MaterialTheme.typography.titleMedium)
        Text("Descripción", style = MaterialTheme.typography.bodyMedium)
    }
}

// ElevatedCard — con sombra más visible
// OutlinedCard — con borde sin sombra
```

### 2.4 Top App Bar

```kotlin
@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun MiPantalla() {
    Scaffold(
        topBar = {
            TopAppBar(
                title = { Text("Mis Productos") },
                navigationIcon = {
                    IconButton(onClick = { /* navegar atrás */ }) {
                        Icon(Icons.AutoMirrored.Filled.ArrowBack, "Atrás")
                    }
                },
                actions = {
                    IconButton(onClick = { /* buscar */ }) {
                        Icon(Icons.Filled.Search, "Buscar")
                    }
                },
                colors = TopAppBarDefaults.topAppBarColors(
                    containerColor = MaterialTheme.colorScheme.primaryContainer
                )
            )
        },
        floatingActionButton = {
            FloatingActionButton(onClick = { /* agregar */ }) {
                Icon(Icons.Filled.Add, "Agregar")
            }
        }
    ) { paddingValues ->
        // Contenido de la pantalla
        LazyColumn(contentPadding = paddingValues) { /* items */ }
    }
}
```

### 2.5 Listas con LazyColumn

```kotlin
// Datos de prueba
data class Producto(val id: Int, val nombre: String, val precio: String, val categoria: String)

val productos = listOf(
    Producto(1, "Laptop Dell XPS 15", "S/ 5,200", "Tecnología"),
    // ...
)

@Composable
fun ListaProductos(productos: List<Producto>) {
    LazyColumn(
        contentPadding = PaddingValues(16.dp),
        verticalArrangement = Arrangement.spacedBy(8.dp)
    ) {
        items(productos, key = { it.id }) { producto ->
            ProductoItem(producto = producto)
        }
    }
}

@Composable
fun ProductoItem(producto: Producto) {
    Card(modifier = Modifier.fillMaxWidth()) {
        ListItem(
            headlineContent = { Text(producto.nombre) },
            supportingContent = { Text(producto.categoria) },
            trailingContent = { 
                Text(producto.precio, style = MaterialTheme.typography.labelLarge)
            },
            leadingContent = {
                Icon(Icons.Filled.ShoppingCart, contentDescription = null)
            }
        )
    }
}
```

---

## 3. Tipografía de Material Design 3 *(10 min)*

M3 define una escala tipográfica con 5 niveles × 3 tamaños:

| Estilo | Uso típico |
|---|---|
| `displayLarge/Medium/Small` | Títulos enormes, pantallas de bienvenida |
| `headlineLarge/Medium/Small` | Títulos de pantalla, encabezados de sección |
| `titleLarge/Medium/Small` | Títulos de cards, dialogs |
| `bodyLarge/Medium/Small` | Texto de contenido principal |
| `labelLarge/Medium/Small` | Botones, etiquetas, badges |

```kotlin
Text("Título de pantalla", style = MaterialTheme.typography.headlineMedium)
Text("Contenido del card", style = MaterialTheme.typography.bodyMedium)
Text("ETIQUETA", style = MaterialTheme.typography.labelLarge)
```

---

## 4. Accesibilidad — WCAG 2.2 en Android *(15 min)*

### 4.1 ¿Por qué es obligatorio?

- **1,300 millones de personas** viven con alguna discapacidad en el mundo ([OMS, 2023](https://www.who.int/news-room/fact-sheets/detail/disability-and-health)).
- Google Play puede rechazar apps que no cumplan mínimos de accesibilidad.
- WCAG 2.2 AA es el estándar legal en múltiples países.

### 4.2 Reglas de accesibilidad para Android

**Contraste mínimo:**
- Texto normal: ratio 4.5:1 mínimo.
- Texto grande (18sp+): ratio 3:1 mínimo.
- M3 con dynamic colors cumple esto automáticamente.

**Tamaño mínimo de toque:**
```kotlin
// Mínimo 48dp × 48dp para cualquier elemento interactivo
IconButton(
    onClick = { },
    modifier = Modifier.size(48.dp)  // no reducir por debajo de esto
) {
    Icon(Icons.Filled.Delete, contentDescription = "Eliminar producto")
}
```

**Content Description para lectores de pantalla (TalkBack):**
```kotlin
// Obligatorio para iconos sin texto visible
Icon(
    imageVector = Icons.Filled.Favorite,
    contentDescription = "Agregar a favoritos"  // TalkBack leerá esto
)

// null solo si es puramente decorativo
Icon(
    imageVector = Icons.Filled.Circle,
    contentDescription = null  // decorativo, TalkBack lo ignora
)
```

**Semánticas personalizadas:**
```kotlin
Box(
    modifier = Modifier.semantics {
        contentDescription = "Producto: Laptop Dell, precio S/ 5200, disponible"
        role = Role.Button
    }
)
```

---

## 5. Performance — Compose Recomposición *(10 min)*

### 5.1 ¿Qué es la recomposición?

Compose re-ejecuta las funciones `@Composable` cuando el estado cambia. La recomposición **innecesaria** desperdicia CPU y batería.

**Principios para minimizar recomposiciones:**

```kotlin
// ❌ MAL: leer state en un scope amplio fuerza recomposición del padre
@Composable
fun PantallaIneficiente() {
    val contador by remember { mutableIntStateOf(0) }
    Text("Pantalla completa")
    Text("Contador: $contador")  // toda la función se recompone cuando cambia
}

// ✅ BIEN: aislar la lectura del state en el composable más pequeño posible
@Composable
fun PantallaEficiente(contador: Int) {
    Text("Pantalla completa")
    ContadorDisplay(contador)  // solo este se recompone
}

@Composable
fun ContadorDisplay(contador: Int) {
    Text("Contador: $contador")
}
```

### 5.2 Baseline Profiles

Los Baseline Profiles le dicen al compilador AOT de Android qué código compilar antes del primer uso:

- **Resultado**: inicio de app 40% más rápido, primeras interacciones 30% más fluidas.
- Se generan automáticamente con el plugin `androidx.benchmark:benchmark-baseline-profile`.
- Fuente: [developer.android.com/baseline-profiles](https://developer.android.com/topic/performance/baselineprofiles).

---

## 6. Resumen de la Sesión

| Concepto | Clave a recordar |
|---|---|
| Material You | Colores dinámicos desde el wallpaper. Usa roles, no colores fijos. |
| Tema dinámico | `dynamicLightColorScheme()` en Android 12+; paleta estática en versiones anteriores |
| Componentes M3 | Button tiene 5 variantes según énfasis. Card, TextField, TopAppBar con Scaffold. |
| Accesibilidad | 48dp mínimo de toque, contraste 4.5:1, `contentDescription` en todos los iconos interactivos |
| Recomposición | Aislar lecturas de estado en el composable más pequeño posible |
| Baseline Profiles | Mejora 40% el startup time — obligatorio en apps de producción |

---

## Tarea Previa al CodLab

1. Genera una paleta de colores para tu proyecto en [m3.material.io/theme-builder](https://m3.material.io/theme-builder) y exporta el código Kotlin.
2. Verifica tu app del CodLab N°2 con TalkBack activado (Configuración → Accesibilidad → TalkBack).

---

## Recursos Adicionales

| Recurso | Enlace | Para qué |
|---|---|---|
| Material Design 3 Docs | [m3.material.io](https://m3.material.io/) | Referencia completa del design system |
| Material Theme Builder | [m3.material.io/theme-builder](https://m3.material.io/theme-builder) | Generar paletas de color personalizadas |
| Compose Component Catalog | [developer.android.com/jetpack/compose/components](https://developer.android.com/jetpack/compose/components) | Todos los componentes disponibles |
| Baseline Profiles | [developer.android.com/baselineprofiles](https://developer.android.com/topic/performance/baselineprofiles) | Mejora de rendimiento en startup |
| OMS — Discapacidad | [who.int/disability](https://www.who.int/news-room/fact-sheets/detail/disability-and-health) | Estadísticas de accesibilidad global |
| WCAG 2.2 | [w3.org/TR/WCAG22](https://www.w3.org/TR/WCAG22/) | Estándar de accesibilidad web/móvil |
