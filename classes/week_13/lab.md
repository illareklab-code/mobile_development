# CodLab N°11 — Auditoría de Seguridad + Firebase ML Kit
**Curso:** Desarrollo de Sistemas Móviles | **Semana:** 13  
**Evaluación:** Individual

---

## Objetivos

- Realizar una auditoría de seguridad del proyecto del curso.
- Corregir las vulnerabilidades OWASP encontradas.
- Integrar Firebase ML Kit para reconocimiento de texto (OCR).
- Configurar Network Security Config para forzar HTTPS.
- Habilitar R8/ProGuard en el build de release.

---

## Parte 1 — Auditoría de Seguridad del Proyecto (60 min)

### Paso 1: Buscar credenciales hardcodeadas

```bash
# Ejecutar desde la raíz del proyecto
grep -rn "api_key\|password\|secret\|token\|private_key" \
    --include="*.kt" --include="*.java" --include="*.xml" \
    --exclude-dir=".git" --exclude-dir="build" .
```

Si encuentras credenciales en el código:
1. Muévalas a `local.properties` (ya está en `.gitignore`).
2. Accédelas desde `build.gradle.kts` con `BuildConfig`.
3. Haz `git log --all --full-history -- local.properties` para confirmar que nunca se commiteó.

### Paso 2: Verificar Firebase Security Rules

Abre Firebase Console → Firestore → Rules. Verifica que tus reglas NO sean:

```javascript
// ❌ ABIERTO — cualquiera puede leer y escribir
allow read, write: if true;
```

Deben ser al menos:
```javascript
// ✅ Solo usuarios autenticados acceden a sus propios datos
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /users/{userId}/{document=**} {
      allow read, write: if request.auth != null && request.auth.uid == userId;
    }
  }
}
```

Prueba las reglas con el **Rules Playground** en Firebase Console.

### Paso 3: Configurar Network Security Config

Crea `res/xml/network_security_config.xml`:

```xml
<?xml version="1.0" encoding="utf-8"?>
<network-security-config>
    <base-config cleartextTrafficPermitted="false">
        <trust-anchors>
            <certificates src="system"/>
        </trust-anchors>
    </base-config>
    <!-- Solo para emulador/desarrollo local -->
    <domain-config cleartextTrafficPermitted="true">
        <domain includeSubdomains="false">10.0.2.2</domain>
        <domain includeSubdomains="false">localhost</domain>
    </domain-config>
</network-security-config>
```

Agrega en `AndroidManifest.xml`:
```xml
<application
    android:networkSecurityConfig="@xml/network_security_config"
    ...>
```

### Paso 4: Habilitar R8 en release

```kotlin
// app/build.gradle.kts
android {
    buildTypes {
        release {
            isMinifyEnabled = true
            isShrinkResources = true
            proguardFiles(
                getDefaultProguardFile("proguard-android-optimize.txt"),
                "proguard-rules.pro"
            )
        }
    }
}
```

Crea un build de release y verifica que compila: `./gradlew assembleRelease`

### Paso 5: Auditoría de permisos

Revisa `AndroidManifest.xml`. Para cada permiso, responde: **¿Realmente lo uso?**

```xml
<!-- Permisos comunes — verifica cada uno -->
<uses-permission android:name="android.permission.INTERNET" />         <!-- ¿Hago peticiones de red? -->
<uses-permission android:name="android.permission.CAMERA" />           <!-- ¿Uso la cámara? -->
<uses-permission android:name="android.permission.READ_MEDIA_IMAGES" /> <!-- ¿Leo imágenes de la galería? -->
```

Elimina cualquier permiso que no uses activamente.

### Paso 6: Completar el Security Checklist

Llena este checklist y preséntalo en el lab:

```
SECURITY CHECKLIST — [Tu nombre] — [Nombre del proyecto]

[ ] No hay credenciales hardcodeadas en el código
[ ] .gitignore incluye local.properties, google-services.json local
[ ] Firebase Security Rules protegen datos por usuario autenticado
[ ] Network Security Config prohíbe HTTP en producción
[ ] R8/ProGuard habilitado en release build
[ ] Solo los permisos Android necesarios están declarados
[ ] Tokens JWT se renuevan automáticamente (interceptor)
[ ] No hay passwords ni tokens en los logs (Logcat)
[ ] android:debuggable no está forzado a true en el manifest
[ ] Dependencias actualizadas (sin CVEs conocidos)

HALLAZGOS ENCONTRADOS:
1. [Describir el problema encontrado]  → CORREGIDO / PENDIENTE
2. ...

RIESGO RESIDUAL (vulnerabilidades que no puedes corregir ahora y por qué):
...
```

---

## Parte 2 — Firebase ML Kit: Reconocimiento de Texto (40 min)

### Paso 7: Agregar dependencia de ML Kit

```kotlin
// app/build.gradle.kts
dependencies {
    // ML Kit Text Recognition (modelo incluido en el APK)
    implementation("com.google.mlkit:text-recognition:16.0.1")
    implementation("com.google.mlkit:text-recognition-latin:16.0.1")

    // CameraX para captura de imágenes
    val cameraxVersion = "1.4.0"
    implementation("androidx.camera:camera-camera2:$cameraxVersion")
    implementation("androidx.camera:camera-lifecycle:$cameraxVersion")
    implementation("androidx.camera:camera-view:$cameraxVersion")
}
```

### Paso 8: ViewModel de OCR

```kotlin
// ui/ocr/OcrViewModel.kt
@HiltViewModel
class OcrViewModel @Inject constructor() : ViewModel() {

    data class OcrUiState(
        val textoReconocido: String = "",
        val isProcessing: Boolean = false,
        val error: String? = null
    )

    private val _uiState = MutableStateFlow(OcrUiState())
    val uiState: StateFlow<OcrUiState> = _uiState.asStateFlow()

    private val recognizer = TextRecognition.getClient(TextRecognizerOptions.DEFAULT_OPTIONS)

    fun procesarImagen(bitmap: Bitmap) {
        viewModelScope.launch {
            _uiState.update { it.copy(isProcessing = true, error = null) }
            val image = InputImage.fromBitmap(bitmap, 0)
            recognizer.process(image)
                .addOnSuccessListener { visionText ->
                    _uiState.update { it.copy(textoReconocido = visionText.text, isProcessing = false) }
                }
                .addOnFailureListener { e ->
                    _uiState.update { it.copy(error = e.message, isProcessing = false) }
                }
        }
    }

    fun limpiar() = _uiState.update { OcrUiState() }
}
```

### Paso 9: Screen de OCR con selección de imagen

```kotlin
@Composable
fun OcrScreen(viewModel: OcrViewModel = hiltViewModel()) {
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()
    var imagenSeleccionada by remember { mutableStateOf<Bitmap?>(null) }

    // Launcher para seleccionar imagen de la galería
    val launcher = rememberLauncherForActivityResult(
        ActivityResultContracts.GetContent()
    ) { uri ->
        uri?.let { selectedUri ->
            // Convertir URI a Bitmap
            val source = ImageDecoder.createSource(LocalContext.current.contentResolver, selectedUri)
            imagenSeleccionada = ImageDecoder.decodeBitmap(source).copy(Bitmap.Config.ARGB_8888, false)
            imagenSeleccionada?.let { viewModel.procesarImagen(it) }
        }
    }

    Scaffold(topBar = { TopAppBar(title = { Text("Reconocimiento de Texto (OCR)") }) }) { padding ->
        Column(
            modifier = Modifier.fillMaxSize().padding(padding).padding(16.dp),
            verticalArrangement = Arrangement.spacedBy(16.dp)
        ) {
            // Imagen seleccionada
            Box(
                modifier = Modifier
                    .fillMaxWidth()
                    .height(200.dp)
                    .background(MaterialTheme.colorScheme.surfaceVariant, RoundedCornerShape(12.dp))
                    .clickable { launcher.launch("image/*") },
                contentAlignment = Alignment.Center
            ) {
                if (imagenSeleccionada != null) {
                    Image(
                        bitmap = imagenSeleccionada!!.asImageBitmap(),
                        contentDescription = "Imagen seleccionada",
                        modifier = Modifier.fillMaxSize().clip(RoundedCornerShape(12.dp)),
                        contentScale = ContentScale.Fit
                    )
                } else {
                    Column(horizontalAlignment = Alignment.CenterHorizontally) {
                        Icon(Icons.Filled.Image, contentDescription = null, modifier = Modifier.size(48.dp))
                        Text("Toca para seleccionar una imagen", style = MaterialTheme.typography.bodyMedium)
                    }
                }
            }

            // Botones
            Row(horizontalArrangement = Arrangement.spacedBy(8.dp)) {
                Button(onClick = { launcher.launch("image/*") }, modifier = Modifier.weight(1f)) {
                    Text("Seleccionar imagen")
                }
                if (uiState.textoReconocido.isNotEmpty()) {
                    OutlinedButton(onClick = viewModel::limpiar, modifier = Modifier.weight(1f)) {
                        Text("Limpiar")
                    }
                }
            }

            // Estado de procesamiento
            if (uiState.isProcessing) {
                Row(verticalAlignment = Alignment.CenterVertically) {
                    CircularProgressIndicator(modifier = Modifier.size(24.dp))
                    Spacer(Modifier.width(8.dp))
                    Text("Procesando imagen...")
                }
            }

            // Texto reconocido
            if (uiState.textoReconocido.isNotEmpty()) {
                Text("Texto reconocido:", style = MaterialTheme.typography.titleSmall)
                OutlinedTextField(
                    value = uiState.textoReconocido,
                    onValueChange = {},
                    modifier = Modifier.fillMaxWidth().heightIn(min = 100.dp),
                    readOnly = true,
                    label = { Text("Texto extraído de la imagen") }
                )
                // Botón de copiar
                OutlinedButton(
                    onClick = {
                        // Copiar al portapapeles
                    },
                    modifier = Modifier.fillMaxWidth()
                ) {
                    Icon(Icons.Filled.ContentCopy, contentDescription = null)
                    Spacer(Modifier.width(8.dp))
                    Text("Copiar texto")
                }
            }

            uiState.error?.let { error ->
                Text("Error: $error", color = MaterialTheme.colorScheme.error)
            }
        }
    }
}
```

---

## Entregables

1. Security Checklist completado con al menos 8/10 ítems marcados como cumplidos.
2. Firebase Security Rules actualizadas y probadas con Rules Playground.
3. Network Security Config configurado (HTTP bloqueado excepto localhost).
4. Feature de OCR funcionando: seleccionar imagen → extraer texto → mostrar en pantalla.
5. R8 habilitado — build de release compila exitosamente.

## Criterios de Evaluación

| Criterio | Puntaje |
|---|---|
| Security Checklist completado con hallazgos documentados | 2 pts |
| Firebase Security Rules protegen datos por usuario | 2 pts |
| Network Security Config prohíbe HTTP en producción | 1 pt |
| OCR funciona: imagen → texto extraído correctamente | 3 pts |
| R8 habilitado: `assembleRelease` compila sin errores | 1 pt |
| No hay credenciales en el código fuente | 1 pt |
| **Total** | **10 pts** |
