# Semana 5 — Diseño UI con Figma y Diseño Responsivo 2026
**Curso:** Desarrollo de Sistemas Móviles | **Unidad:** U2  
**Sesión:** Presencial | **Duración:** 2 horas

---

## Objetivos de la Sesión

- Usar Figma como herramienta de diseño colaborativo y prototyping.
- Aplicar Design Tokens para automatizar la coherencia entre diseño y código.
- Diseñar interfaces responsivas para Android: teléfono, tablet y foldable.
- Usar WindowSizeClass API de Compose para layouts adaptativos.
- Definir el proyecto del curso en equipos.

---

## 1. Figma — La Herramienta Estándar de Diseño en 2026 *(25 min)*

### 1.1 ¿Por qué Figma?

| Herramienta | Estado en 2026 | Razón |
|---|---|---|
| **Figma** | ✅ Estándar de industria | Colaborativo, basado en web, integración con devs |
| Adobe XD | ❌ Descontinuado | Adobe lo cerró en enero 2024 |
| Sketch | ❌ Obsoleto | Solo Mac, ecosistema cerrado |
| InVision | ❌ Cerrado | Servicio detenido en 2024 |
| Mockflow / Wireframe.cc | ❌ Para wireframes simples | No aptos para diseño de producción |

**Características clave de Figma en 2026:**
- **Colaboración en tiempo real**: múltiples diseñadores y desarrolladores trabajando simultáneamente.
- **Componentes y variantes**: diseña una vez, reutiliza en todo el proyecto.
- **Auto Layout**: layouts flexibles que se adaptan al contenido (equivalente a `Column`/`Row` en Compose).
- **Variables (Design Tokens)**: define colores, espaciado y tipografía como variables sincronizadas con código.
- **Dev Mode**: los desarrolladores ven el código CSS/Android, medidas exactas en dp/sp y pueden copiar assets.
- **FigJam**: pizarra colaborativa para user flows, wireframes y design thinking.
- **Prototyping**: conecta pantallas para crear flujos interactivos sin código.

### 1.2 Conceptos fundamentales de Figma

**Frames**: equivalentes a pantallas o contenedores. Configura para Mobile (390×844 px = iPhone 14 Pro) o Android (360×800 dp).

**Componentes (Atomic Design)**:
```
Átomos    → Button, TextField, Icon, Badge
Moléculas → SearchBar (TextField + Icon + Button)
Organismos→ ProductCard (Image + Text + Badge + Button)
Templates  → ListScreen (TopAppBar + SearchBar + lista de Cards)
Páginas    → App completa con todas las pantallas
```

**Auto Layout** (equivalente a Column/Row/FlexBox):
- Dirección: horizontal o vertical.
- Gap: espacio entre elementos (`Arrangement.spacedBy()` en Compose).
- Padding: espacio interior (`Modifier.padding()` en Compose).
- Fill container / Fixed: `Modifier.fillMaxWidth()` vs `Modifier.width(X.dp)`.

### 1.3 Design Tokens

Los Design Tokens son variables de diseño que se usan tanto en Figma como en el código:

```
// En Figma (Variables):
color/primary = #6750A4
color/secondary = #958DA5
spacing/small = 8
spacing/medium = 16
spacing/large = 24
typography/headline = Inter 28sp SemiBold

// En código Kotlin (Theme.kt) — los mismos valores:
val md_theme_light_primary = Color(0xFF6750A4)
val spacingSmall = 8.dp
val spacingMedium = 16.dp
val spacingLarge = 24.dp
```

**Plugin recomendado:** Token Studio for Figma (sincroniza tokens entre Figma y código automáticamente).

**Herramienta oficial:** Material Theme Builder ([m3.material.io/theme-builder](https://m3.material.io/theme-builder)) exporta tokens directamente a Kotlin.

---

## 2. Handoff — Del Diseño al Código *(15 min)*

### 2.1 ¿Cómo leer un diseño de Figma como desarrollador?

En **Dev Mode** de Figma (activar en la barra superior):
- Clic en cualquier elemento → ver propiedades de código a la derecha.
- Las medidas se muestran en dp (Android) o pt (iOS).
- Los colores muestran el hex y el nombre del token.
- Puedes copiar el código Compose directamente (beta en 2026).

**Proceso de handoff:**
1. Diseñador crea componentes con Auto Layout en Figma.
2. Exporta assets (SVG → VectorDrawable, PNG → diferentes densidades).
3. Desarrollador inspecciona en Dev Mode:
   - Padding y márgenes en dp.
   - Tipografía: tamaño en sp, peso, interlineado.
   - Colores: hex → buscar el rol M3 equivalente.
4. Desarrollador implementa en Compose, comparando con el diseño.

---

## 3. Diseño Responsivo para Android 2026 *(30 min)*

### 3.1 La fragmentación de pantallas en Android

Android corre en miles de dispositivos con pantallas muy diferentes:

| Tipo | Tamaño típico | Ejemplo |
|---|---|---|
| Teléfono pequeño | 5" — 360×640 dp | Galaxy A05 |
| Teléfono estándar | 6" — 390×844 dp | Pixel 8 |
| Teléfono grande | 6.7" — 412×915 dp | Galaxy S25 Ultra |
| Tablet pequeña | 8" — 600×960 dp | iPad mini equivalente |
| Tablet estándar | 11" — 800×1280 dp | Galaxy Tab S9 |
| Foldable (plegado) | ~6.2" — 360×820 dp | Galaxy Z Fold 6 (cerrado) |
| Foldable (abierto) | ~7.6" — 840×1010 dp | Galaxy Z Fold 6 (abierto) |

### 3.2 WindowSizeClass — Los 3 breakpoints oficiales

Google define 3 clases de tamaño de ventana:

```kotlin
// Compact  < 600dp  → diseño de teléfono (columna única)
// Medium   600-840dp → diseño de tablet pequeña (2 columnas posibles)
// Expanded > 840dp  → diseño de tablet/foldable (lista-detalle side by side)
```

Implementación en Compose:

```kotlin
@Composable
fun AppAdaptiva() {
    val windowSizeClass = calculateWindowSizeClass(LocalContext.current as Activity)

    when (windowSizeClass.widthSizeClass) {
        WindowWidthSizeClass.Compact -> {
            // Teléfono: navegación bottom bar
            AppConBottomNavigation()
        }
        WindowWidthSizeClass.Medium -> {
            // Tablet pequeña: navegación rail lateral
            AppConNavigationRail()
        }
        WindowWidthSizeClass.Expanded -> {
            // Tablet grande / foldable abierto: navigation drawer + lista-detalle
            AppConNavigationDrawer()
        }
    }
}

// Lista-detalle para tablets
@Composable
fun ListaDetalleScreen(
    windowSizeClass: WindowWidthSizeClass,
    productos: List<Producto>,
    productoSeleccionado: Producto?
) {
    if (windowSizeClass == WindowWidthSizeClass.Expanded) {
        // Tablet: ambos paneles visibles simultáneamente
        Row(modifier = Modifier.fillMaxSize()) {
            ListaPanel(productos, modifier = Modifier.weight(0.4f))
            DetallePanel(productoSeleccionado, modifier = Modifier.weight(0.6f))
        }
    } else {
        // Teléfono: navegación secuencial
        if (productoSeleccionado == null) ListaPanel(productos)
        else DetallePanel(productoSeleccionado)
    }
}
```

### 3.3 Foldables — 5 estados de pantalla

```
Estado 1: FLAT          — completamente abierto (tablet)
Estado 2: HALF_OPENED   — doblado en ángulo (libro abierto)
Estado 3: BOOK          — orientado verticalmente semiabierto
Estado 4: TENT          — como carpa (apoyado en borde)
Estado 5: CLOSED        — como teléfono normal

// Detectar el estado con Jetpack Window Manager
val foldingFeature: FoldingFeature? = ...
when (foldingFeature?.state) {
    FoldingFeature.State.HALF_OPENED -> mostrarLayoutDesdeDoblez()
    FoldingFeature.State.FLAT        -> mostrarLayoutCompleto()
    null                             -> mostrarLayoutTeléfono()
}
```

### 3.4 Multi-Window Support

Android 7+ permite apps en modo split-screen y picture-in-picture:

```kotlin
// Detectar si la app está en modo multi-window
val isInMultiWindow = LocalContext.current.isInMultiWindowMode

// PiP (Picture-in-Picture) — para video players, llamadas
override fun onUserLeaveHint() {
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
        enterPictureInPictureMode(PictureInPictureParams.Builder().build())
    }
}
```

---

## 4. Micro-interacciones y Estados de UI *(15 min)*

### 4.1 Los 4 estados que toda pantalla debe contemplar

```
┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐
│   LOADING   │  │   SUCCESS   │  │    ERROR    │  │    EMPTY    │
│             │  │             │  │             │  │             │
│ ⟳ Cargando  │  │ [Contenido] │  │ ⚠ Error     │  │ 📭 No hay   │
│             │  │             │  │  "Reintentar"│  │  elementos  │
└─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘
```

```kotlin
sealed class UiState<out T> {
    data object Loading : UiState<Nothing>()
    data class Success<T>(val data: T) : UiState<T>()
    data class Error(val message: String, val canRetry: Boolean = true) : UiState<Nothing>()
    data object Empty : UiState<Nothing>()
}

@Composable
fun <T> EstadoUI(
    state: UiState<T>,
    onRetry: () -> Unit = {},
    content: @Composable (T) -> Unit
) {
    when (state) {
        is UiState.Loading -> Box(Modifier.fillMaxSize(), contentAlignment = Alignment.Center) {
            CircularProgressIndicator()
        }
        is UiState.Success -> content(state.data)
        is UiState.Error   -> ErrorScreen(state.message, state.canRetry, onRetry)
        is UiState.Empty   -> EmptyScreen()
    }
}
```

---

## 5. Presentación del Proyecto del Curso *(15 min)*

### 5.1 Estructura del proyecto

El proyecto del curso se desarrolla en 3 sprints:

| Sprint | Semana | Entregable |
|---|---|---|
| Sprint 1 | 1–7 | App base: UI, Figma, MVVM, 1 API REST, Firebase | 30% |
| Sprint 2 | 9–11 | Backend + BD local + Autenticación | 40% N2 |
| Sprint 3 | 13–15 | Seguridad + Tests + CI/CD | 40% N2 |
| Exposición Final | 16 | App completa funcional | 30% |

### 5.2 Requisitos técnicos del proyecto

- Kotlin + Jetpack Compose (sin XML)
- MVVM + Clean Architecture (UseCase + Repository)
- Hilt para inyección de dependencias
- Jetpack Navigation con type-safe routes
- Mínimo 5 pantallas
- Firebase Firestore o API REST propia
- Room para persistencia local
- Firebase Authentication
- GitHub con historial de commits limpio
- CI/CD con GitHub Actions (Sprint 3)

---

## 6. Resumen de la Sesión

| Concepto | Clave a recordar |
|---|---|
| Figma | Herramienta estándar. Frames, Auto Layout, Componentes, Variables. Dev Mode para handoff. |
| Design Tokens | Variables de diseño compartidas entre Figma y código. M3 Theme Builder las exporta. |
| Responsivo | 3 clases: Compact (<600dp), Medium (600-840dp), Expanded (>840dp) |
| WindowSizeClass | Bottom Nav en Compact, Navigation Rail en Medium, Drawer + panel en Expanded |
| Foldables | 5 estados físicos. Usar Jetpack Window Manager para detectar postura. |
| Estados de UI | Loading, Success, Error, Empty — toda pantalla debe manejar los 4 |

---

## Tarea Previa al CodLab

1. Crea una cuenta gratuita en [figma.com](https://figma.com) con tu correo institucional.
2. Mira el tutorial "Figma for Beginners" en el canal oficial de Figma en YouTube (4 videos, ~30 min total).
3. Define con tu equipo el concepto de app para el proyecto del curso.

---

## Recursos Adicionales

| Recurso | Enlace | Para qué |
|---|---|---|
| Figma | [figma.com](https://figma.com) | Herramienta de diseño |
| Material Theme Builder | [m3.material.io/theme-builder](https://m3.material.io/theme-builder) | Generar tema y tokens |
| Window Size Classes | [developer.android.com/guide/topics/large-screens/support-different-screen-sizes](https://developer.android.com/guide/topics/large-screens/support-different-screen-sizes) | Diseño adaptativo oficial |
| Foldables Guide | [developer.android.com/guide/topics/large-screens/learn-about-foldables](https://developer.android.com/guide/topics/large-screens/learn-about-foldables) | Soporte de dispositivos plegables |
| Android Form Factors | [developer.android.com/large-screens](https://developer.android.com/large-screens) | Guía oficial multi-formato |
