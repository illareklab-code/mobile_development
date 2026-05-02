# Semana 13 — Seguridad Móvil: OWASP Top 10 y Firebase ML Kit
**Curso:** Desarrollo de Sistemas Móviles | **Unidad:** U4  
**Sesión:** Presencial | **Duración:** 2 horas

---

## Objetivos de la Sesión

- Identificar las 10 vulnerabilidades más críticas en apps móviles según OWASP 2024.
- Implementar mitigaciones concretas para cada vulnerabilidad.
- Aplicar certificate pinning con OkHttp contra ataques MITM.
- Integrar Firebase ML Kit para funcionalidades de IA on-device.
- Configurar análisis de seguridad con SonarQube y Dependabot.

---

## 1. OWASP Mobile Top 10 — 2024 Edition *(40 min)*

### ¿Qué es OWASP?

El Open Web Application Security Project ([OWASP](https://owasp.org)) publica rankings de vulnerabilidades de seguridad actualizados. La versión 2016 está completamente obsoleta — en 2026 usamos la de 2024. Fuente: [OWASP Mobile Top 10 2024](https://owasp.org/www-project-mobile-top-10/).

---

### M1: Improper Credential Usage

**Problema:** credenciales hardcodeadas en el código fuente.

```kotlin
// ❌ NUNCA hacer esto
val API_KEY = "sk-abc123secreto"  // visible en el APK reverse-engineered

// ✅ Usar local.properties (no commiteado a Git)
// local.properties:
// API_KEY=sk-abc123secreto

// build.gradle.kts:
val apiKey = properties.getProperty("API_KEY") ?: throw GradleException("API_KEY not set")
buildConfigField("String", "API_KEY", "\"$apiKey\"")

// Código Kotlin:
val key = BuildConfig.API_KEY  // inyectado en compilación, ofuscado con R8
```

---

### M2: Inadequate Supply Chain Security

**Problema:** dependencias de terceros con vulnerabilidades conocidas.

```kotlin
// ✅ Dependabot: crea PRs automáticos cuando hay actualizaciones de seguridad
// .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: "gradle"
    directory: "/"
    schedule:
      interval: "weekly"
    security-updates-only: true
```

---

### M3: Insecure Authentication/Authorization

**Problema:** tokens JWT sin expiración, sesiones que no se invalidan.

```kotlin
// ✅ Siempre verificar que el token no esté expirado
// Firebase Auth maneja esto automáticamente — el token dura 1 hora
// Para APIs propias: access token (15 min) + refresh token (7 días)

// Interceptor de Retrofit que renueva el token automáticamente
class AuthInterceptor @Inject constructor(
    private val tokenRepository: TokenRepository
) : Interceptor {
    override fun intercept(chain: Interceptor.Chain): Response {
        val token = tokenRepository.getAccessToken()
        val response = chain.proceed(
            chain.request().newBuilder()
                .header("Authorization", "Bearer $token")
                .build()
        )
        if (response.code == 401) {
            // Renovar token y reintentar
            val newToken = tokenRepository.refresh()
            return chain.proceed(
                chain.request().newBuilder()
                    .header("Authorization", "Bearer $newToken")
                    .build()
            )
        }
        return response
    }
}
```

---

### M4: Insufficient Input/Output Validation

**Problema:** datos del usuario no validados → inyección SQL via API, XSS en WebViews.

```kotlin
// ✅ Validar siempre en el backend (nunca confiar solo en el cliente)
// En el app: también validar para mejor UX (no esperar al servidor)

// ❌ Peligroso: abrir URLs arbitrarias en WebView
webView.loadUrl(intent.getStringExtra("url")!!)  // puede cargar contenido malicioso

// ✅ Validar la URL antes de cargarla
fun loadSafeUrl(url: String) {
    if (url.startsWith("https://miapp.com/")) {
        webView.loadUrl(url)
    } else {
        // rechazar URL sospechosa
    }
}

// ✅ Deshabilitar JavaScript si no es necesario
webView.settings.javaScriptEnabled = false
```

---

### M5: Insecure Communication

**Problema:** tráfico HTTP sin cifrar, TLS 1.0/1.1, sin certificate pinning.

```kotlin
// ✅ Network Security Config — forzar HTTPS
// res/xml/network_security_config.xml
```
```xml
<?xml version="1.0" encoding="utf-8"?>
<network-security-config>
    <!-- Desactivar HTTP para todas las conexiones de producción -->
    <base-config cleartextTrafficPermitted="false">
        <trust-anchors>
            <certificates src="system"/>
        </trust-anchors>
    </base-config>
    <!-- Solo permitir HTTP para debug en localhost -->
    <debug-overrides>
        <trust-anchors>
            <certificates src="system"/>
            <certificates src="user"/>
        </trust-anchors>
    </debug-overrides>
</network-security-config>
```

```kotlin
// ✅ Certificate Pinning — rechazar cualquier certificado que no sea el nuestro
val certificatePinner = CertificatePinner.Builder()
    .add("api.miapp.com", "sha256/HASH_DEL_CERTIFICADO=")
    .build()

OkHttpClient.Builder().certificatePinner(certificatePinner).build()
```

---

### M6: Inadequate Privacy Controls

**Problema:** solicitar permisos innecesarios, guardar PII sin cifrar, datos en logs.

```kotlin
// ❌ Nunca loguear datos sensibles
Log.d("Auth", "Password: ${user.password}")  // visible en logcat de cualquier app

// ✅ Usar ProGuard/R8 para eliminar logs en release
// proguard-rules.pro
// -assumenosideeffects class android.util.Log { *; }

// ✅ Solicitar solo permisos necesarios con explicación al usuario
// AndroidManifest.xml: SOLO los permisos que realmente necesitas
```

---

### M7: Insufficient Binary Protections

**Problema:** APK puede reverse-engineered fácilmente.

```kotlin
// app/build.gradle.kts
android {
    buildTypes {
        release {
            isMinifyEnabled = true   // habilitar R8/ProGuard
            isShrinkResources = true // eliminar recursos no usados
            proguardFiles(getDefaultProguardFile("proguard-android-optimize.txt"), "proguard-rules.pro")
        }
    }
}
```

---

### M8: Security Misconfiguration

**Problema:** Firebase Rules abiertas, debuggable=true en producción.

```xml
<!-- ❌ NUNCA en producción -->
<application android:debuggable="true" ...>

<!-- ✅ Correcto — Android Studio maneja esto automáticamente en release builds -->
<application android:debuggable="false" ...>
```

```javascript
// ❌ Firebase Security Rules abiertas (como vienen por defecto)
match /{document=**} {
  allow read, write: if true;  // CUALQUIERA puede leer/escribir TODO
}

// ✅ Reglas correctas
match /usuarios/{userId}/{document=**} {
  allow read, write: if request.auth != null && request.auth.uid == userId;
}
```

---

### M9: Insecure Data Storage

**Problema:** guardar datos sensibles en SharedPreferences sin cifrar, en archivos externos.

```kotlin
// ❌ Guardar contraseña en SharedPreferences
sharedPreferences.edit().putString("password", userPassword).apply()

// ✅ Usar Android Keystore para datos criptográficos
val keyGenerator = KeyGenerator.getInstance(KeyProperties.KEY_ALGORITHM_AES, "AndroidKeyStore")
keyGenerator.init(
    KeyGenParameterSpec.Builder(
        "MiKeyAlias",
        KeyProperties.PURPOSE_ENCRYPT or KeyProperties.PURPOSE_DECRYPT
    )
    .setBlockModes(KeyProperties.BLOCK_MODE_GCM)
    .setEncryptionPaddings(KeyProperties.ENCRYPTION_PADDING_NONE)
    .build()
)
// La clave nunca sale del hardware seguro
```

---

### M10: Insufficient Cryptography

**Problema:** usar MD5/SHA1, cifrado ECB, claves hardcodeadas.

```kotlin
// ❌ MD5 está roto — no usar para nada de seguridad
MessageDigest.getInstance("MD5")

// ✅ Usar SHA-256 mínimo para hashes, AES-256-GCM para cifrado
MessageDigest.getInstance("SHA-256")

// ✅ Para contraseñas: bcrypt, Argon2 o PBKDF2 (en el servidor, nunca en el cliente)
```

---

## 2. Threat Modeling — STRIDE *(10 min)*

STRIDE es un framework para identificar amenazas antes de implementar:

| Letra | Amenaza | Ejemplo móvil |
|---|---|---|
| **S**poofing | Hacerse pasar por otro usuario | Robo de token JWT |
| **T**ampering | Modificar datos en tránsito | MITM modificando respuesta de API |
| **R**epudiation | Negar haber realizado una acción | Log de auditoría inexistente |
| **I**nformation Disclosure | Exponer información sensible | API key en el código fuente |
| **D**enial of Service | Tumbar la app o el servidor | Rate limiting no implementado |
| **E**levation of Privilege | Obtener más permisos de los debidos | Acceder a datos de otro usuario |

---

## 3. Firebase ML Kit en 2026 *(25 min)*

ML Kit permite integrar IA **on-device** (sin internet) en tu app Android.

### 3.1 Modelos disponibles en ML Kit

| Feature | Descripción | Casos de uso |
|---|---|---|
| **Text Recognition v2** | OCR — extraer texto de imágenes | Escáner de documentos, recibos |
| **Barcode Scanning** | Leer códigos QR, EAN, UPC | Control de inventario, pagos |
| **Face Detection** | Detectar rostros, expresiones | Filtros AR, verificación de identidad |
| **Object Detection** | Identificar objetos en imágenes | Clasificación de productos |
| **Language ID** | Detectar el idioma de un texto | Apps multiidioma automáticas |
| **Translation** | Traducción on-device (17 idiomas) | Chat sin internet |
| **Pose Detection** | Detectar postura del cuerpo | Apps de fitness, fisioterapia |

### 3.2 Text Recognition — Implementación

```kotlin
// app/build.gradle.kts
implementation("com.google.mlkit:text-recognition:16.0.1")
implementation("com.google.mlkit:text-recognition-latin:16.0.1")  // modelo para español/inglés

// Función de reconocimiento de texto
fun reconocerTexto(imageBitmap: Bitmap, onResult: (String) -> Unit, onError: (Exception) -> Unit) {
    val image = InputImage.fromBitmap(imageBitmap, 0)
    val recognizer = TextRecognition.getClient(TextRecognizerOptions.DEFAULT_OPTIONS)

    recognizer.process(image)
        .addOnSuccessListener { visionText ->
            onResult(visionText.text)  // texto completo reconocido
        }
        .addOnFailureListener { e ->
            onError(e)
        }
}
```

### 3.3 Barcode Scanner — Implementación

```kotlin
implementation("com.google.mlkit:barcode-scanning:17.3.0")

fun escanearQR(imageBitmap: Bitmap, onResult: (String) -> Unit) {
    val image = InputImage.fromBitmap(imageBitmap, 0)
    val scanner = BarcodeScanning.getClient(
        BarcodeScannerOptions.Builder()
            .setBarcodeFormats(Barcode.FORMAT_QR_CODE, Barcode.FORMAT_EAN_13)
            .build()
    )

    scanner.process(image)
        .addOnSuccessListener { barcodes ->
            barcodes.firstOrNull()?.rawValue?.let { onResult(it) }
        }
}
```

---

## 4. SonarQube y Análisis de Calidad *(10 min)*

SonarQube 10.x analiza automáticamente el código en busca de:
- **Code Smells:** código funcional pero difícil de mantener.
- **Bugs:** código con alta probabilidad de causar errores.
- **Vulnerabilidades:** problemas de seguridad (hardcoded secrets, SQL injection patterns).
- **Security Hotspots:** código que requiere revisión manual de seguridad.

```yaml
# .github/workflows/sonar.yml
- name: SonarQube Analysis
  run: ./gradlew sonar
  env:
    SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
    SONAR_HOST_URL: https://sonarcloud.io
```

---

## 5. Resumen de la Sesión

| Vulnerabilidad | Mitigación clave |
|---|---|
| M1 Credentials | BuildConfig, no hardcoding, `.gitignore` para `local.properties` |
| M2 Supply Chain | Dependabot, verificar licencias y reputación de dependencias |
| M3 Auth/Authz | Tokens con expiración, refresh automático, Firebase Auth |
| M5 Communication | HTTPS siempre, Network Security Config, certificate pinning |
| M7 Binary | R8/ProGuard en release builds, `isMinifyEnabled = true` |
| M8 Misconfiguration | Firebase Rules por usuario, no debuggable en prod |
| M9 Data Storage | Android Keystore para claves, DataStore cifrado para preferencias |
| ML Kit | OCR, QR, traducción on-device — sin internet, sin costo de API |

---

## Recursos Adicionales

| Recurso | Enlace | Para qué |
|---|---|---|
| OWASP Mobile Top 10 2024 | [owasp.org/www-project-mobile-top-10](https://owasp.org/www-project-mobile-top-10/) | Referencia de vulnerabilidades |
| Firebase ML Kit | [developers.google.com/ml-kit](https://developers.google.com/ml-kit) | Features de IA on-device |
| Network Security Config | [developer.android.com/training/articles/security-config](https://developer.android.com/training/articles/security-config) | Configurar HTTPS |
| Android Keystore | [developer.android.com/training/articles/keystore](https://developer.android.com/training/articles/keystore) | Almacenamiento seguro de claves |
| IBM Data Breach Report 2024 | [ibm.com/reports/data-breach](https://www.ibm.com/reports/data-breach) | Costo de vulnerabilidades |
