# CodLab N°2 — Fundamentos de Kotlin y Jetpack Compose
**Curso:** Desarrollo de Sistemas Móviles | **Semana:** 2  
**Evaluación:** Individual

---

## Objetivos

- Aplicar los fundamentos de Kotlin: funciones, nulabilidad, lambdas, data classes.
- Crear composables básicos con Jetpack Compose.
- Manejar estado local con `remember` y `mutableStateOf`.
- Implementar el patrón de state hoisting.
- Construir una app de contador funcional con Material Design 3.

## Prerrequisitos

- CodLab N°1 completado (Android Studio instalado y corriendo).
- Haber leído la teoría de la Semana 2.

## Herramientas Requeridas

- Android Studio Meerkat
- Emulador Pixel 9 Pro API 35 (configurado en CodLab N°1)

---

## Paso 1: Crear nuevo proyecto

Crea un proyecto nuevo llamado `KotlinComposeBasics` con las mismas configuraciones del CodLab N°1 (Kotlin DSL, API 24, Empty Activity).

---

## Paso 2: Practicar Kotlin en un archivo separado

Crea el archivo `app/src/main/kotlin/.../KotlinBasics.kt`:

```kotlin
package com.tuapellido.kotlincomposebasics

// --- FUNCIONES ---
fun sumar(a: Int, b: Int): Int = a + b

fun saludar(nombre: String, idioma: String = "es"): String {
    return when (idioma) {
        "es" -> "Hola, $nombre"
        "en" -> "Hello, $nombre"
        else -> "Hi, $nombre"
    }
}

// Función de extensión
fun String.esPalindromo(): Boolean = this == this.reversed()

// --- DATA CLASS ---
data class Tarea(
    val id: Int,
    val titulo: String,
    val completada: Boolean = false,
    val prioridad: Prioridad = Prioridad.MEDIA
)

enum class Prioridad { ALTA, MEDIA, BAJA }

// --- SEALED CLASS para estado ---
sealed class ResultadoOperacion<out T> {
    data object Cargando : ResultadoOperacion<Nothing>()
    data class Exito<T>(val datos: T) : ResultadoOperacion<T>()
    data class Error(val mensaje: String) : ResultadoOperacion<Nothing>()
}

// --- LAMBDAS Y COLECCIONES ---
fun demostrarColecciones() {
    val tareas = listOf(
        Tarea(1, "Aprender Kotlin", completada = true, prioridad = Prioridad.ALTA),
        Tarea(2, "Hacer CodLab", completada = false, prioridad = Prioridad.ALTA),
        Tarea(3, "Leer docs de Compose", completada = false, prioridad = Prioridad.MEDIA),
        Tarea(4, "Ver tutorial de Hilt", completada = true, prioridad = Prioridad.BAJA)
    )

    // filter: devuelve lista filtrada
    val pendientes = tareas.filter { !it.completada }
    println("Pendientes: ${pendientes.size}") // 2

    // map: transforma cada elemento
    val titulos = tareas.map { it.titulo }
    println("Títulos: $titulos")

    // sortedBy: ordena por campo
    val porPrioridad = tareas.sortedBy { it.prioridad }

    // groupBy: agrupa por campo
    val porEstado = tareas.groupBy { it.completada }
    val completadas = porEstado[true] ?: emptyList()

    // any / all / count
    val hayAlta = tareas.any { it.prioridad == Prioridad.ALTA } // true
    val todasCompletadas = tareas.all { it.completada }          // false
    val nCompletadas = tareas.count { it.completada }            // 2
}
```

---

## Paso 3: App de Contador con State Hoisting

Reemplaza el contenido de `MainActivity.kt`:

```kotlin
package com.tuapellido.kotlincomposebasics

import android.os.Bundle
import androidx.activity.ComponentActivity
import androidx.activity.compose.setContent
import androidx.compose.foundation.layout.*
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.unit.dp
import com.tuapellido.kotlincomposebasics.ui.theme.KotlinComposeBasicsTheme

class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContent {
            KotlinComposeBasicsTheme {
                Surface(modifier = Modifier.fillMaxSize()) {
                    ContadorScreen()
                }
            }
        }
    }
}

// --- SCREEN: coordina el estado ---
// State hoisting: el estado vive aquí, se pasa hacia abajo como parámetros
@Composable
fun ContadorScreen() {
    var contador by remember { mutableIntStateOf(0) }
    var historial by remember { mutableStateOf(listOf<String>()) }

    ContadorContent(
        contador = contador,
        historial = historial,
        onIncrementar = {
            contador++
            historial = historial + ["Incrementado a $contador"]
        },
        onDecrementar = {
            if (contador > 0) {
                contador--
                historial = historial + ["Decrementado a $contador"]
            }
        },
        onReiniciar = {
            contador = 0
            historial = emptyList()
        }
    )
}

// --- CONTENT: solo recibe datos y callbacks, no tiene estado propio ---
@Composable
fun ContadorContent(
    contador: Int,
    historial: List<String>,
    onIncrementar: () -> Unit,
    onDecrementar: () -> Unit,
    onReiniciar: () -> Unit,
    modifier: Modifier = Modifier
) {
    Column(
        modifier = modifier
            .fillMaxSize()
            .padding(24.dp),
        horizontalAlignment = Alignment.CenterHorizontally,
        verticalArrangement = Arrangement.spacedBy(16.dp)
    ) {
        Text(
            text = "Contador",
            style = MaterialTheme.typography.headlineLarge
        )

        // Número del contador con color dinámico según valor
        Text(
            text = "$contador",
            style = MaterialTheme.typography.displayLarge,
            color = when {
                contador > 10 -> MaterialTheme.colorScheme.error
                contador > 5  -> MaterialTheme.colorScheme.primary
                else          -> MaterialTheme.colorScheme.onSurface
            }
        )

        // Botones
        Row(horizontalArrangement = Arrangement.spacedBy(8.dp)) {
            OutlinedButton(onClick = onDecrementar, enabled = contador > 0) {
                Text("−")
            }
            Button(onClick = onIncrementar) {
                Text("+")
            }
        }

        TextButton(onClick = onReiniciar) {
            Text("Reiniciar")
        }

        // Historial de acciones
        if (historial.isNotEmpty()) {
            HorizontalDivider()
            Text("Historial", style = MaterialTheme.typography.titleMedium)
            historial.takeLast(5).reversed().forEach { entrada ->
                Text(
                    text = "• $entrada",
                    style = MaterialTheme.typography.bodySmall,
                    color = MaterialTheme.colorScheme.secondary
                )
            }
        }
    }
}
```

---

## Paso 4: Agregar Preview

```kotlin
@Preview(showBackground = true, showSystemUi = true)
@Composable
fun ContadorScreenPreview() {
    KotlinComposeBasicsTheme {
        ContadorContent(
            contador = 7,
            historial = listOf("Incrementado a 5", "Incrementado a 6", "Incrementado a 7"),
            onIncrementar = {},
            onDecrementar = {},
            onReiniciar = {}
        )
    }
}
```

---

## Paso 5: Agregar lista de tareas (extensión)

Agrega una segunda pantalla con una lista de tareas usando los tipos del Paso 2:

```kotlin
@Composable
fun TareasScreen(modifier: Modifier = Modifier) {
    val tareas = remember {
        mutableStateListOf(
            Tarea(1, "Aprender Kotlin", completada = true, prioridad = Prioridad.ALTA),
            Tarea(2, "Hacer CodLab N°2", prioridad = Prioridad.ALTA),
            Tarea(3, "Leer docs de Compose", prioridad = Prioridad.MEDIA)
        )
    }

    LazyColumn(
        modifier = modifier.fillMaxSize().padding(16.dp),
        verticalArrangement = Arrangement.spacedBy(8.dp)
    ) {
        item {
            Text("Mis Tareas", style = MaterialTheme.typography.headlineMedium)
            Spacer(Modifier.height(8.dp))
        }
        items(tareas, key = { it.id }) { tarea ->
            TareaItem(
                tarea = tarea,
                onToggle = { 
                    val index = tareas.indexOf(tarea)
                    tareas[index] = tarea.copy(completada = !tarea.completada)
                }
            )
        }
    }
}

@Composable
fun TareaItem(tarea: Tarea, onToggle: () -> Unit) {
    Card(modifier = Modifier.fillMaxWidth()) {
        Row(
            modifier = Modifier.padding(16.dp),
            verticalAlignment = Alignment.CenterVertically
        ) {
            Checkbox(checked = tarea.completada, onCheckedChange = { onToggle() })
            Spacer(Modifier.width(8.dp))
            Column(modifier = Modifier.weight(1f)) {
                Text(
                    text = tarea.titulo,
                    style = MaterialTheme.typography.bodyLarge,
                    color = if (tarea.completada) 
                        MaterialTheme.colorScheme.outline 
                    else 
                        MaterialTheme.colorScheme.onSurface
                )
                Text(
                    text = tarea.prioridad.name,
                    style = MaterialTheme.typography.labelSmall,
                    color = when (tarea.prioridad) {
                        Prioridad.ALTA  -> MaterialTheme.colorScheme.error
                        Prioridad.MEDIA -> MaterialTheme.colorScheme.primary
                        Prioridad.BAJA  -> MaterialTheme.colorScheme.secondary
                    }
                )
            }
        }
    }
}
```

---

## Entregables

1. App corriendo en emulador con el contador funcional (incrementar, decrementar, reiniciar).
2. Historial de acciones visible.
3. Lista de tareas con checkbox funcional (marcar/desmarcar).
4. Preview funcionando para al menos un composable.
5. Código en repositorio Git (misma rama del CodLab N°1).

## Criterios de Evaluación

| Criterio | Puntaje |
|---|---|
| Contador funcional (incrementar, decrementar, reiniciar) | 3 pts |
| State hoisting correcto (estado en Screen, no en Content) | 2 pts |
| Color dinámico del contador según valor | 1 pt |
| Lista de tareas con `mutableStateListOf` y `copy()` | 2 pts |
| Data classes y sealed classes usados correctamente | 1 pt |
| Preview implementado | 1 pt |
| **Total** | **10 pts** |
