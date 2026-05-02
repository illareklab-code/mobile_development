# CodLab N°1 — Configuración de Android Studio y Primera App
**Curso:** Desarrollo de Sistemas Móviles | **Semana:** 1  
**Evaluación:** Individual

---

## Objetivos

- Instalar y configurar Android Studio Meerkat correctamente.
- Crear un proyecto Android con Jetpack Compose usando Kotlin DSL.
- Comprender la estructura de un proyecto Android moderno.
- Ejecutar la app en emulador y dispositivo físico.
- Hacer la primera modificación de UI con Jetpack Compose.

## Prerrequisitos

- Computadora con mínimo 8 GB RAM (16 GB recomendado), 10 GB espacio libre.
- JDK 21 (se instala automáticamente con Android Studio Meerkat).
- Cuenta de Google (para el emulador con Google Play Services).

## Herramientas Requeridas

- Android Studio Meerkat (descarga: [developer.android.com/studio](https://developer.android.com/studio))
- Emulador con Pixel 9 Pro, API 35 (Vanilla Ice Cream)
- Cable USB (si usas dispositivo físico) con depuración USB habilitada

---

## Paso 1: Instalación de Android Studio

1. Descarga Android Studio Meerkat desde [developer.android.com/studio](https://developer.android.com/studio).
2. Ejecuta el instalador y acepta la configuración estándar (Standard Setup).
3. Descarga los componentes: Android SDK, Android Emulator, SDK Build-Tools.
4. Al finalizar, abre Android Studio y completa el wizard inicial.

**Verificar instalación correcta:**
- En el menú: *Help → About* → debe mostrar Android Studio Meerkat | 2024.3.x
- En *SDK Manager* → Android 15.0 (API 35) debe estar instalado.

---

## Paso 2: Crear el emulador

1. Ve a *Device Manager* (ícono en la barra lateral derecha o `View → Tool Windows → Device Manager`).
2. Clic en **+** → *Create Virtual Device*.
3. Selecciona: **Pixel 9 Pro** → *Next*.
4. Selecciona imagen del sistema: **API Level 35** (Vanilla Ice Cream), arquitectura x86_64 → *Download* si no está instalada.
5. Nombre: `Pixel_9_Pro_API35` → *Finish*.
6. Inicia el emulador con el botón ▶ — espera que arranque completamente.

---

## Paso 3: Crear el proyecto

1. En la pantalla de bienvenida: **New Project**.
2. Selecciona: **Empty Activity** (usa Jetpack Compose por defecto) → *Next*.
3. Configura:

| Campo | Valor |
|---|---|
| Name | `HelloWorldApp` |
| Package name | `com.tuapellido.helloworldapp` |
| Save location | carpeta de tu elección |
| Language | **Kotlin** |
| Minimum SDK | **API 24** ("Android 7.0 Nougat") |
| Build configuration language | **Kotlin DSL (build.gradle.kts)** ← importante |

4. Clic en **Finish** y espera a que Gradle sincronice (puede tomar 2-5 min la primera vez).

> **Nota:** Asegúrate de seleccionar **Kotlin DSL** (archivo `.kts`), NO Groovy (`.gradle`). Groovy es la opción legacy y no se usará en este curso.

---

## Paso 4: Explorar la estructura del proyecto

Navega en el panel izquierdo (*Project* view en modo **Android**) y observa:

```
app/
├── manifests/
│   └── AndroidManifest.xml       ← Declara actividades, permisos, app metadata
├── kotlin+java/
│   └── com.tuapellido.helloworldapp/
│       └── MainActivity.kt       ← Punto de entrada + UI en Compose
└── res/
    ├── drawable/                  ← Íconos e imágenes vectoriales
    ├── mipmap/                    ← Íconos de lanzador (launcher icon)
    └── values/
        ├── colors.xml             ← Paleta de colores (referenciada desde el tema)
        └── themes.xml             ← Tema Material Design 3

Gradle Scripts/
├── build.gradle.kts (Project)    ← Configuración del proyecto raíz
└── build.gradle.kts (Module :app)← Dependencias y configuración del módulo
```

Abre `app/build.gradle.kts` y observa:
```kotlin
android {
    compileSdk = 35

    defaultConfig {
        applicationId = "com.tuapellido.helloworldapp"
        minSdk = 24
        targetSdk = 35
        versionCode = 1
        versionName = "1.0"
    }
}

dependencies {
    implementation("androidx.activity:activity-compose:1.9.x")
    implementation(platform("androidx.compose:compose-bom:2024.xx.xx"))
    implementation("androidx.compose.ui:ui")
    implementation("androidx.compose.material3:material3")
}
```

---

## Paso 5: Ejecutar la app

1. Asegúrate de que el emulador está corriendo (aparece en el selector de dispositivo arriba).
2. Clic en el botón ▶ **Run 'app'** (Shift+F10).
3. Espera a que compile y se instale en el emulador.
4. Debes ver: pantalla con el texto **"Hello Android!"** en el centro.

**Si tienes dispositivo físico:**
- Activa *Opciones de Desarrollador* en tu teléfono (toca "Número de compilación" 7 veces).
- Habilita *Depuración USB*.
- Conecta el cable y acepta la autorización en el teléfono.
- Selecciona tu dispositivo en el selector de dispositivo de Android Studio.

---

## Paso 6: Modificar la UI con Jetpack Compose

Abre `MainActivity.kt`. Verás código como este:

```kotlin
class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContent {
            HelloWorldAppTheme {
                Greeting(name = "Android")
            }
        }
    }
}

@Composable
fun Greeting(name: String, modifier: Modifier = Modifier) {
    Text(
        text = "Hello $name!",
        modifier = modifier
    )
}
```

**Ejercicio: Modifica la UI para que muestre tu nombre y la carrera:**

```kotlin
@Composable
fun Greeting(name: String, career: String, modifier: Modifier = Modifier) {
    Column(
        modifier = modifier
            .fillMaxSize()
            .padding(24.dp),
        horizontalAlignment = Alignment.CenterHorizontally,
        verticalArrangement = Arrangement.Center
    ) {
        Text(
            text = "Hola, soy $name",
            style = MaterialTheme.typography.headlineMedium
        )
        Spacer(modifier = Modifier.height(8.dp))
        Text(
            text = career,
            style = MaterialTheme.typography.bodyLarge,
            color = MaterialTheme.colorScheme.secondary
        )
        Spacer(modifier = Modifier.height(16.dp))
        Text(
            text = "Semana 1 — Desarrollo de Sistemas Móviles",
            style = MaterialTheme.typography.labelMedium
        )
    }
}
```

Actualiza la llamada en `setContent`:
```kotlin
Greeting(name = "Tu Nombre", career = "Ingeniería de Sistemas")
```

Usa el botón **🔄 (Apply Changes)** para ver el resultado sin recompilar todo.

---

## Paso 7: Usar el Preview

Agrega esta función al final del archivo para previsualizar sin ejecutar el emulador:

```kotlin
@Preview(showBackground = true, showSystemUi = true)
@Composable
fun GreetingPreview() {
    HelloWorldAppTheme {
        Greeting(name = "Preview User", career = "Ingeniería de Sistemas")
    }
}
```

Clic en el panel **Split** (arriba a la derecha del editor) para ver código y preview simultáneamente.

---

## Paso 8: Commit inicial en Git

Android Studio integra Git. Configúralo para este proyecto:

1. *VCS → Create Git Repository* → selecciona la carpeta del proyecto.
2. Agrega un archivo `.gitignore` (ya generado por Android Studio — verifica que incluye `/build` y `/.idea`).
3. *VCS → Commit* → selecciona todos los archivos → mensaje: `feat: initial project setup`.

---

## Entregables

Al final del laboratorio presenta:

1. App corriendo en el emulador con tu nombre y carrera visibles.
2. Preview funcionando en Android Studio.
3. Código fuente en un repositorio Git (GitHub) — comparte el link.

## Criterios de Evaluación

| Criterio | Puntaje |
|---|---|
| Android Studio instalado correctamente (versión Meerkat) | 2 pts |
| Proyecto creado con Kotlin DSL y API 24/35 | 2 pts |
| App corre en emulador sin errores | 3 pts |
| UI modificada muestra nombre y carrera del estudiante | 2 pts |
| Repositorio Git con commit inicial | 1 pt |
| **Total** | **10 pts** |
