# Semana 2 — Frameworks de Desarrollo Móvil y Fundamentos de Kotlin
**Curso:** Desarrollo de Sistemas Móviles | **Unidad:** U1  
**Sesión:** Presencial | **Duración:** 2 horas

---

## Objetivos de la Sesión

- Comparar los principales frameworks de desarrollo móvil en 2026.
- Justificar la elección de Kotlin + Jetpack Compose como stack principal del curso.
- Dominar los conceptos fundamentales de Kotlin: funciones, nulabilidad, lambdas.
- Comprender la arquitectura Clean + MVVM como estándar de la industria.

---

## 1. Panorama de Frameworks Móviles en 2026 *(30 min)*

### 1.1 ¿Qué framework elegir?

En 2026 hay más opciones que nunca para desarrollar apps móviles. La elección depende del contexto del proyecto:

| Framework | Lenguaje | Plataformas | Rendimiento | Curva de aprendizaje | Ideal para |
|---|---|---|---|---|---|
| **Kotlin + Compose** (Nativo Android) | Kotlin | Android | ⭐⭐⭐⭐⭐ | Media | Apps Android premium, acceso total a hardware |
| **Swift + SwiftUI** (Nativo iOS) | Swift | iOS/macOS/visionOS | ⭐⭐⭐⭐⭐ | Media | Apps iOS premium, ecosistema Apple |
| **Flutter 4.x** | Dart | Android, iOS, Web, Desktop, Embedded | ⭐⭐⭐⭐ | Media | Un código base para múltiples plataformas |
| **Kotlin Multiplatform (KMM)** | Kotlin | Android + iOS (lógica compartida) | ⭐⭐⭐⭐⭐ | Alta | Equipos que quieren UIs nativas con lógica compartida |
| **React Native** | JavaScript/TypeScript | Android + iOS | ⭐⭐⭐ | Baja (si sabes JS) | Equipos web que migran a móvil |
| **.NET MAUI** | C# | Android, iOS, Windows, macOS | ⭐⭐⭐ | Media (si sabes C#) | Equipos .NET/Microsoft |

> **Estadística 2024:** Kotlin es el lenguaje #1 para desarrollo Android. Flutter es el framework multiplataforma más popular según [Stack Overflow Developer Survey 2024](https://survey.stackoverflow.co/2024/technology#most-popular-technologies-misc-tech).

### 1.2 Kotlin en el mercado

- Declarado **lenguaje oficial de Android** por Google en 2017.
- **95% del código nuevo en Android** está escrito en Kotlin (fuente: [Google Android Developers Blog](https://android-developers.googleblog.com/)).
- Kotlin ocupa el puesto **#7 de lenguajes más usados** globalmente ([JetBrains Developer Ecosystem 2024](https://www.jetbrains.com/lp/devecosystem-2024/)).
- **Kotlin Multiplatform** llega a v2.0 estable en 2024: compartir lógica de negocio entre Android e iOS sin perder UIs nativas.

### 1.3 Flutter 4.x — La alternativa multiplataforma

Flutter, mantenido por Google, compila a código nativo ARM y usa su propio motor de renderizado (Impeller en 2026). Características clave:

- **Un código base** → Android, iOS, Web, Desktop, Embedded.
- **Hot Reload**: ver cambios en < 1 segundo sin recompilar.
- **Dart 3.x**: lenguaje con null safety, pattern matching, records.
- **Flutter 4.x + IA**: integración nativa con Gemini API y on-device ML.
- **Riverpod** como el sistema de state management estándar en la comunidad Flutter.

### 1.4 React Native — La opción para equipos web

React Native permite usar JavaScript/TypeScript con componentes nativos. En 2026:

- **Nueva Arquitectura** (JSI + Fabric): elimina el bridge antiguo, mejor rendimiento.
- **Hermes Engine**: motor JS optimizado para móvil (startup más rápido).
- Popular en startups con equipos web que necesitan una app móvil rápidamente.
- Limitación: acceso al hardware más complejo que nativo, dependencia del ecosistema JS.

### 1.5 ¿Por qué este curso usa Kotlin + Jetpack Compose?

1. **Demanda laboral**: el 80% de ofertas de trabajo en desarrollo Android en Perú y Latinoamérica piden Kotlin.
2. **Rendimiento**: código nativo, sin bridges ni abstracciones adicionales.
3. **Soporte de Google**: actualizaciones, documentación y herramientas de primera clase.
4. **Composable multiplataforma**: con KMM puedes reutilizar lógica en iOS sin cambiar de lenguaje.
5. **Jetpack Compose** es el futuro de la UI en Android — Google ya no invierte en el sistema de vistas XML.

---

## 2. Kotlin 2.0+ — El Lenguaje *(40 min)*

### 2.1 Características fundamentales

#### Tipado estático con inferencia de tipos
```kotlin
val nombre: String = "Android"  // tipo explícito
val version = 2.0               // tipo inferido: Double
var contador = 0                // var = mutable, val = inmutable
```

#### Funciones
```kotlin
// Función regular
fun saludar(nombre: String): String {
    return "Hola, $nombre"
}

// Función de expresión (cuerpo de una línea)
fun cuadrado(n: Int) = n * n

// Parámetros con valor por defecto
fun crearUsuario(nombre: String, rol: String = "estudiante") = "$nombre ($rol)"

// Funciones de extensión — añaden comportamiento a clases existentes sin heredar
fun String.capitalizeFirst(): String = 
    replaceFirstChar { if (it.isLowerCase()) it.titlecase() else it.toString() }

// Uso: "hola mundo".capitalizeFirst() → "Hola mundo"
```

#### Nulabilidad — el sistema de null safety de Kotlin
```kotlin
var nombre: String = "Kotlin"     // no puede ser null
var apodo: String? = null          // puede ser null (tipo nullable)

// Operador safe call (?.)
val longitud = apodo?.length       // null si apodo es null

// Operador Elvis (?:) — valor por defecto si es null
val longitud = apodo?.length ?: 0

// Operador de aserción (!!) — EVITAR en producción, lanza NullPointerException
val longitud = apodo!!.length      // explota si apodo es null

// Let — ejecutar código solo si no es null
apodo?.let { valor ->
    println("El apodo tiene ${valor.length} caracteres")
}
```

#### Data classes — modelos de datos
```kotlin
data class Producto(
    val id: Int,
    val nombre: String,
    val precio: Double,
    val disponible: Boolean = true
)

// Características automáticas de data class:
// - equals() / hashCode()
// - toString() → "Producto(id=1, nombre=Laptop, precio=2500.0, disponible=true)"
// - copy() → crear una copia con cambios puntuales
val producto = Producto(1, "Laptop", 2500.0)
val descuento = producto.copy(precio = 2000.0)
```

#### Expresiones lambda y funciones de orden superior
```kotlin
// Lambda: función sin nombre, asignada a una variable
val suma: (Int, Int) -> Int = { a, b -> a + b }
suma(3, 4)  // → 7

// Funciones de colección con lambdas (muy usadas en Kotlin)
val numeros = listOf(1, 2, 3, 4, 5)

val pares = numeros.filter { it % 2 == 0 }           // [2, 4]
val dobles = numeros.map { it * 2 }                   // [2, 4, 6, 8, 10]
val suma = numeros.reduce { acc, n -> acc + n }       // 15
val texto = numeros.joinToString(separator = " | ")  // "1 | 2 | 3 | 4 | 5"
```

#### Sealed classes — estados finitos y exhaustivos
```kotlin
// Sealed class: todos los subtipos están en el mismo archivo
// Usado para modelar estados de UI
sealed class UiState<out T> {
    data object Loading : UiState<Nothing>()
    data class Success<T>(val data: T) : UiState<T>()
    data class Error(val message: String) : UiState<Nothing>()
}

// El compilador garantiza que el when es exhaustivo (no necesita else)
when (state) {
    is UiState.Loading -> mostrarCargando()
    is UiState.Success -> mostrarDatos(state.data)
    is UiState.Error   -> mostrarError(state.message)
}
```

### 2.2 Coroutines — Concurrencia en Kotlin *(introducción)*

Las **Coroutines** son la forma nativa de manejar código asíncrono en Kotlin. En 2026, reemplazaron completamente a AsyncTask, RxJava y callbacks anidados.

```kotlin
// Problema sin coroutines: callback hell
apiService.getUser(id) { user ->
    apiService.getPosts(user.id) { posts ->
        apiService.getComments(posts.first().id) { comments ->
            // código imposible de mantener...
        }
    }
}

// Con coroutines: código legible, secuencial
suspend fun cargarDatos() {
    val user = apiService.getUser(id)       // suspende, no bloquea el hilo
    val posts = apiService.getPosts(user.id)
    val comments = apiService.getComments(posts.first().id)
    // continúa aquí cuando todo esté listo
}
```

**Dispatchers** — hilos de ejecución:
```kotlin
withContext(Dispatchers.IO)   // operaciones de red y base de datos
withContext(Dispatchers.Main) // actualizar la UI (hilo principal)
withContext(Dispatchers.Default) // cómputo intensivo (CPU)
```

---

## 3. Clean Architecture + MVVM en Android *(20 min)*

### 3.1 ¿Por qué arquitectura?

Sin arquitectura, los proyectos se convierten en "código espagueti":
- Toda la lógica en `MainActivity` → difícil de testear y mantener.
- Cambios en la UI rompen la lógica de negocio.
- Imposible trabajar en equipo sin conflictos.

### 3.2 La arquitectura estándar 2026

```
┌─────────────────────────────────────────┐
│           UI LAYER (Compose)            │
│  Screen → ViewModel → UiState (Flow)   │
├─────────────────────────────────────────┤
│          DOMAIN LAYER (Opcional)        │
│  UseCase: GetProductsUseCase            │
│  → encapsula 1 operación de negocio     │
├─────────────────────────────────────────┤
│            DATA LAYER                   │
│  Repository → [RemoteDataSource]        │
│                [LocalDataSource/Room]   │
└─────────────────────────────────────────┘
```

**Unidirectional Data Flow (UDF):**
```
Usuario click → ViewModel.onEvent() → actualiza StateFlow → Compose recompone UI
```

Las **dependencias** siempre apuntan hacia adentro: UI depende de Domain, Domain no depende de nada externo.

### 3.3 Jetpack Compose 1.7+

**Compose es una revolución en la forma de construir UIs Android:**

| Approach antiguo (XML) | Compose (actual) |
|---|---|
| XML layouts separados del código | UI como funciones Kotlin (`@Composable`) |
| findViewById + binding manual | Estado reactivo automático |
| Difícil de testear | Fácil testing con Compose Testing API |
| No type-safe | 100% type-safe en Kotlin |
| Recomposición manual | Recomposición automática e inteligente |

---

## 4. Kotlin 2.0 — Novedades 2026 *(10 min)*

- **Compilador K2**: 2x más rápido en builds incrementales.
- **Kotlin Multiplatform stable**: compartir código entre Android, iOS, Web, Desktop.
- **Context Receivers**: pasar contexto implícitamente entre funciones.
- **Guard clauses con when**: sintaxis más expresiva.
- **Power Assert Plugin**: mejores mensajes de error en tests.

---

## 5. Resumen de la Sesión

| Concepto | Clave a recordar |
|---|---|
| Frameworks 2026 | Kotlin nativo > rendimiento; Flutter > multiplataforma; React Native > equipos JS |
| Kotlin vs Java | Kotlin es más conciso, null-safe, con coroutines nativas. Java es legacy en Android. |
| Nulabilidad | `String?` puede ser null. Usa `?.`, `?:`, `let {}` — nunca `!!` en producción |
| Data classes | Modelos de datos con equals, hashCode, toString y copy automáticos |
| Lambdas | Funciones de primera clase: `filter`, `map`, `reduce` son esenciales |
| MVVM + Clean | UI → ViewModel → UseCase → Repository → DataSource |

---

## Tarea Previa al CodLab

1. Completar el tutorial oficial de Kotlin: [play.kotlinlang.org](https://play.kotlinlang.org/) — secciones Hello World, Variables, Functions, Null Safety.
2. Ver el video "Kotlin in 100 seconds" en YouTube.
3. Asegúrate de que tu proyecto del CodLab N°1 compila correctamente.

---

## Recursos Adicionales

| Recurso | Enlace | Para qué |
|---|---|---|
| Kotlin Official Docs | [kotlinlang.org/docs](https://kotlinlang.org/docs/home.html) | Referencia completa del lenguaje |
| Kotlin Playground | [play.kotlinlang.org](https://play.kotlinlang.org/) | Practicar Kotlin en el navegador |
| JetBrains Dev Ecosystem 2024 | [jetbrains.com/lp/devecosystem-2024](https://www.jetbrains.com/lp/devecosystem-2024/) | Estadísticas de uso de Kotlin |
| Compose Docs | [developer.android.com/compose](https://developer.android.com/jetpack/compose) | UI framework del curso |
| Flutter docs | [docs.flutter.dev](https://docs.flutter.dev/) | Comparativa con el alternativo |
| Stack Overflow Survey 2024 | [survey.stackoverflow.co/2024](https://survey.stackoverflow.co/2024/) | Popularidad de lenguajes y frameworks |
