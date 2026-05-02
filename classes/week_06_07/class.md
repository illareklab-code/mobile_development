# Semanas 6–7 — HTTP/REST, Coroutines, Retrofit y Firebase
**Curso:** Desarrollo de Sistemas Móviles | **Unidad:** U2  
**Sesión:** Presencial | **Duración:** 4 horas (2 sesiones)

---

## Objetivos de las Sesiones

- Entender HTTP/3 y los principios REST para el diseño de APIs.
- Implementar peticiones de red con Retrofit + OkHttp + Kotlinx Serialization.
- Manejar concurrencia con Kotlin Coroutines y structured concurrency.
- Implementar el patrón offline-first con Room como caché local.
- Conectar la app a Firebase Firestore con listeners en tiempo real.
- Asegurar las comunicaciones con OAuth 2.1 + PKCE y certificate pinning.

---

## SEMANA 6

## 1. HTTP y Protocolos de Red en 2026 *(20 min)*

### 1.1 Evolución de HTTP

| Versión | Año | Mejora principal |
|---|---|---|
| HTTP/1.1 | 1997 | Conexiones persistentes, pero una solicitud por conexión a la vez |
| HTTP/2 | 2015 | Multiplexing: varias solicitudes por conexión TCP simultáneas |
| **HTTP/3** | **2022–2026** | **Protocolo QUIC (UDP): sin head-of-line blocking, mejor en redes móviles** |

**HTTP/3 en móvil 2026:**
- Construido sobre QUIC (Quick UDP Internet Connections), desarrollado por Google.
- Elimina el "head-of-line blocking" de TCP: si un paquete se pierde, no bloquea los demás.
- Crítico para redes móviles con pérdida de paquetes (4G con mala señal, WiFi con interferencias).
- Adopción: +35% de los sitios top-1000 ya soportan HTTP/3 ([W3Techs HTTP/3 stats](https://w3techs.com/technologies/details/ce-http3)).
- OkHttp 4.11+ soporta HTTP/3 automáticamente cuando el servidor lo tiene habilitado.

### 1.2 Principios REST

REST (Representational State Transfer) es un estilo arquitectónico para APIs:

| Principio | Descripción | Ejemplo |
|---|---|---|
| **Recursos** | Las URLs identifican recursos, no acciones | `/productos`, `/productos/42`, `/usuarios/5/pedidos` |
| **Verbos HTTP** | El verbo indica la acción | GET → leer, POST → crear, PUT/PATCH → actualizar, DELETE → eliminar |
| **Stateless** | Cada petición es independiente | El servidor no guarda el "estado" de la conversación |
| **JSON** | Formato estándar de intercambio | `{"id": 1, "nombre": "Laptop", "precio": 5200.0}` |

**Códigos de estado que debes conocer:**

| Código | Significado | Cuándo usarlo |
|---|---|---|
| 200 OK | Éxito | GET, PUT, PATCH exitoso |
| 201 Created | Creado exitosamente | POST que crea un recurso |
| 204 No Content | Éxito sin cuerpo | DELETE exitoso |
| 400 Bad Request | Error del cliente | Datos inválidos en la petición |
| 401 Unauthorized | No autenticado | Token expirado o ausente |
| 403 Forbidden | No autorizado | Autenticado pero sin permiso |
| 404 Not Found | Recurso no encontrado | ID que no existe |
| 422 Unprocessable | Validación fallida | Datos que no cumplen las reglas |
| 500 Internal Error | Error del servidor | Bug en el backend |

### 1.3 GraphQL 3.0 vs REST

```
REST: múltiples endpoints específicos
  GET /usuarios/42
  GET /usuarios/42/posts
  GET /usuarios/42/posts/5/comentarios

GraphQL: un solo endpoint, el cliente pide exactamente lo que necesita
  POST /graphql
  {
    usuario(id: 42) {
      nombre
      posts(limit: 5) {
        titulo
        comentarios { contenido }
      }
    }
  }
```

**Cuándo usar cada uno:**
- **REST**: cuando tienes CRUD estándar, equipos separados de frontend/backend, APIs públicas.
- **GraphQL**: cuando el frontend necesita datos muy específicos, muchos tipos de clientes (móvil, web, TV).

---

## 2. Kotlin Coroutines — Concurrencia Moderna *(25 min)*

### 2.1 El problema de la concurrencia en Android

En Android, la UI corre en el **Main Thread** (hilo principal). Bloquear el Main Thread con operaciones lentas (red, BD) causa ANR (Application Not Responding):

```kotlin
// ❌ MAL: llamada de red en el Main Thread → ANR después de 5 segundos
class MalaActivity : ComponentActivity() {
    override fun onCreate(...) {
        val json = URL("https://api.example.com/data").readText()  // bloquea UI!
    }
}
```

### 2.2 Structured Concurrency

Las coroutines se organizan en una jerarquía. Si el padre se cancela, todos los hijos se cancelan:

```kotlin
// viewModelScope: atado al ViewModel — se cancela cuando el ViewModel se destruye
// lifecycleScope: atado a la Activity/Fragment
// rememberCoroutineScope(): atado al composable

@HiltViewModel
class ProductosViewModel @Inject constructor(...) : ViewModel() {

    fun cargarDatos() {
        viewModelScope.launch {
            // Este bloque corre en una coroutine
            // Si el ViewModel se destruye, esta coroutine se cancela automáticamente

            val productos = withContext(Dispatchers.IO) {
                // Dispatchers.IO: pool de hilos para operaciones de entrada/salida
                apiService.getProductos()
            }
            // Al volver aquí, estamos en el Main Thread (por defecto en viewModelScope)
            _uiState.value = ProductosUiState.Success(productos)
        }
    }

    fun cargarParalelo() {
        viewModelScope.launch {
            // async + await: ejecutar en paralelo
            val productosDeferred = async(Dispatchers.IO) { apiService.getProductos() }
            val categoriasDeferred = async(Dispatchers.IO) { apiService.getCategorias() }

            // await(): espera el resultado (sin bloquear el hilo)
            val productos = productosDeferred.await()
            val categorias = categoriasDeferred.await()
            // Ambas peticiones corrieron en paralelo → más rápido
        }
    }
}
```

### 2.3 Kotlin Flow — Streams reactivos

```kotlin
// Flow: emite múltiples valores a lo largo del tiempo
// Cold flow: no emite hasta que hay un collector
fun obtenerProductosFlujo(): Flow<List<Producto>> = flow {
    emit(localDb.getProductos())              // primer emit: datos cacheados
    val actualizados = api.getProductos()
    localDb.guardarTodos(actualizados)
    emit(actualizados)                        // segundo emit: datos frescos
}

// StateFlow: siempre tiene valor actual, permite múltiples collectors
// SharedFlow: broadcast a múltiples collectors, configurable

// Operadores de Flow
val productosDisponibles: Flow<List<Producto>> = productosFlow
    .map { productos -> productos.filter { it.disponible } }
    .distinctUntilChanged()  // evita emitir si el valor no cambió

// Combinar dos flows
combine(productosFlow, busquedaFlow) { productos, query ->
    productos.filter { it.nombre.contains(query, ignoreCase = true) }
}
```

---

## 3. Retrofit + OkHttp + Kotlinx Serialization *(25 min)*

### 3.1 Kotlinx Serialization — El moderno sustituto de Gson

```kotlin
// Gson (legacy): no null-safe, no multiplatform
// Moshi: mejor que Gson pero menos integrado con Kotlin
// Kotlinx Serialization (2026): nativo Kotlin, compile-time safe, multiplatform

// build.gradle.kts
implementation("org.jetbrains.kotlinx:kotlinx-serialization-json:1.7.1")
implementation("com.jakewharton.retrofit2:retrofit2-kotlinx-serialization-converter:1.0.0")
```

```kotlin
// Modelo anotado con @Serializable
@Serializable
data class ProductoDto(
    val id: Int,
    val nombre: String,
    val precio: Double,
    @SerialName("imagen_url") val imagenUrl: String,  // mapear nombre JSON diferente
    val disponible: Boolean = true                     // valor por defecto si no viene en JSON
)
```

### 3.2 Retrofit 2.9+

```kotlin
// Define la API como una interfaz Kotlin
interface ProductosApi {
    @GET("v1/productos")
    suspend fun getProductos(): List<ProductoDto>

    @GET("v1/productos/{id}")
    suspend fun getProducto(@Path("id") id: Int): ProductoDto

    @POST("v1/productos")
    suspend fun crearProducto(@Body producto: ProductoDto): ProductoDto

    @PUT("v1/productos/{id}")
    suspend fun actualizarProducto(@Path("id") id: Int, @Body producto: ProductoDto): ProductoDto

    @DELETE("v1/productos/{id}")
    suspend fun eliminarProducto(@Path("id") id: Int)

    @GET("v1/productos")
    suspend fun buscarProductos(@Query("q") query: String): List<ProductoDto>
}
```

### 3.3 OkHttp con Interceptors

```kotlin
val okHttpClient = OkHttpClient.Builder()
    .addInterceptor(
        HttpLoggingInterceptor().apply {
            level = if (BuildConfig.DEBUG) HttpLoggingInterceptor.Level.BODY
                    else HttpLoggingInterceptor.Level.NONE
        }
    )
    .addInterceptor { chain ->
        // Interceptor de autenticación: agrega el token JWT a cada petición
        val token = tokenStore.getAccessToken()
        val request = chain.request().newBuilder()
            .addHeader("Authorization", "Bearer $token")
            .build()
        chain.proceed(request)
    }
    .connectTimeout(30, TimeUnit.SECONDS)
    .readTimeout(30, TimeUnit.SECONDS)
    .build()
```

---

## SEMANA 7

## 4. Seguridad en Comunicaciones *(20 min)*

### 4.1 OAuth 2.1 + PKCE

PKCE (Proof Key for Code Exchange) es obligatorio en OAuth 2.1 para apps móviles:

```
1. App genera code_verifier (string aleatorio 43-128 chars)
2. App genera code_challenge = BASE64URL(SHA256(code_verifier))
3. App abre navegador → servidor de autorización con code_challenge
4. Usuario inicia sesión en el navegador (no en la app)
5. Servidor redirige de vuelta con authorization_code
6. App intercambia authorization_code + code_verifier → access_token
```

**¿Por qué PKCE?** Previene que apps maliciosas intercepten el authorization_code.

### 4.2 Certificate Pinning con OkHttp

```kotlin
// Certificate pinning: la app solo acepta certificados con hashes específicos
// Previene ataques MITM (man-in-the-middle) incluso con certificados raíz comprometidos
val certificatePinner = CertificatePinner.Builder()
    .add("api.miapp.com", "sha256/AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA=")
    .build()

val okHttpClient = OkHttpClient.Builder()
    .certificatePinner(certificatePinner)
    .build()

// Para obtener el hash SHA-256 del certificado:
// openssl s_client -connect api.miapp.com:443 | openssl x509 -pubkey -noout | openssl pkey -pubin -outform der | openssl dgst -sha256 -binary | base64
```

### 4.3 JWT con Refresh Token Rotation

```
Access Token:  corto (15 min) — se envía en cada petición HTTP
Refresh Token: largo (7 días) — se guarda seguro, se usa solo para renovar

Flujo de renovación automática:
1. Petición con Access Token expirado → 401 Unauthorized
2. Interceptor detecta el 401
3. Interceptor llama a POST /auth/refresh con el Refresh Token
4. Servidor devuelve nuevo Access Token + nuevo Refresh Token
5. Interceptor guarda los nuevos tokens
6. Interceptor reintenta la petición original con el nuevo Access Token
```

---

## 5. Patrón Offline-First *(20 min)*

### 5.1 ¿Qué es offline-first?

Una app offline-first funciona correctamente **sin conexión a internet**:
- Lee datos desde una caché local (Room) primero.
- Sincroniza con el servidor cuando hay conexión.
- El usuario nunca ve una pantalla de "sin conexión" — ve datos (posiblemente no actualizados).

```kotlin
// Repository con estrategia Cache-Then-Network
class ProductosRepositoryImpl @Inject constructor(
    private val api: ProductosApi,
    private val dao: ProductoDao
) : ProductosRepository {

    override fun getProductos(): Flow<List<Producto>> = flow {
        // 1. Emitir datos del caché local inmediatamente
        emit(dao.getAll().map { it.toDomain() })

        // 2. Intentar actualizar desde la red
        try {
            val remotos = api.getProductos()
            dao.upsertAll(remotos.map { it.toEntity() })
            emit(dao.getAll().map { it.toDomain() })
        } catch (e: IOException) {
            // Sin conexión — los datos del caché ya fueron emitidos, no hay error para el usuario
        }
    }
}
```

---

## 6. Firebase Firestore *(20 min)*

### 6.1 Estructura de datos en Firestore

Firestore es una base de datos NoSQL de documentos:

```
Firestore
└── colecciones (equivalen a tablas)
    └── productos (colección)
        ├── producto_001 (documento)
        │   ├── nombre: "Laptop Dell XPS"
        │   ├── precio: 5200.0
        │   └── disponible: true
        └── producto_002 (documento)
            ├── nombre: "Auriculares Sony"
            ├── precio: 890.0
            └── disponible: true
```

### 6.2 CRUD con Firestore en Kotlin

```kotlin
// Obtener referencia
val db = Firebase.firestore

// Crear/actualizar documento
suspend fun guardarProducto(producto: Producto) {
    db.collection("productos")
        .document(producto.id)
        .set(producto.toMap())
        .await()
}

// Leer con listener en tiempo real (Flow)
fun getProductos(): Flow<List<Producto>> = callbackFlow {
    val listener = db.collection("productos")
        .addSnapshotListener { snapshot, error ->
            if (error != null) {
                close(error)
                return@addSnapshotListener
            }
            val productos = snapshot?.documents
                ?.mapNotNull { it.toObject<ProductoDto>()?.toDomain() }
                ?: emptyList()
            trySend(productos)
        }
    // Limpiar cuando el Flow se cancela
    awaitClose { listener.remove() }
}

// Eliminar
suspend fun eliminarProducto(id: String) {
    db.collection("productos").document(id).delete().await()
}
```

---

## 7. Caching con Coil 3

Coil 3 es la librería estándar de carga de imágenes en Android 2026:

```kotlin
implementation("io.coil-kt.coil3:coil-compose:3.0.4")
implementation("io.coil-kt.coil3:coil-network-okhttp:3.0.4")

// Uso en Compose
AsyncImage(
    model = "https://example.com/imagen.jpg",
    contentDescription = "Imagen del producto",
    contentScale = ContentScale.Crop,
    modifier = Modifier.fillMaxWidth().height(200.dp),
    placeholder = painterResource(R.drawable.placeholder),
    error = painterResource(R.drawable.error_image)
)
```

---

## 8. Resumen de las Sesiones

| Concepto | Clave a recordar |
|---|---|
| HTTP/3 | QUIC (UDP): sin head-of-line blocking. Mejor en redes móviles con pérdida de paquetes |
| Coroutines | `viewModelScope.launch`, `withContext(Dispatchers.IO)`, `async/await` para paralelo |
| StateFlow | Stream reactivo. `combine()` para juntar múltiples flows |
| Retrofit | Define la API como interfaz Kotlin con `suspend fun`. Kotlinx Serialization, no Gson |
| OkHttp | Interceptors para logging y auth. Certificate pinning contra MITM |
| OAuth 2.1 | PKCE obligatorio. JWT: access (15min) + refresh (7d) con rotación automática |
| Offline-first | Emitir caché → actualizar red → emitir actualizado. Sin error si no hay internet |
| Firestore | `callbackFlow` para listeners en tiempo real. `.await()` para operaciones puntuales |

---

## Recursos Adicionales

| Recurso | Enlace | Para qué |
|---|---|---|
| Kotlin Coroutines Docs | [kotlinlang.org/docs/coroutines-overview](https://kotlinlang.org/docs/coroutines-overview.html) | Referencia completa |
| Retrofit Docs | [square.github.io/retrofit](https://square.github.io/retrofit/) | Configuración y anotaciones |
| OkHttp Docs | [square.github.io/okhttp](https://square.github.io/okhttp/) | Interceptors, certificate pinning |
| Firebase Firestore | [firebase.google.com/docs/firestore](https://firebase.google.com/docs/firestore) | Guía oficial Firestore |
| Coil 3 Docs | [coil-kt.github.io/coil](https://coil-kt.github.io/coil/) | Carga de imágenes |
| HTTP/3 Adoption | [w3techs.com/technologies/details/ce-http3](https://w3techs.com/technologies/details/ce-http3) | Estadísticas de adopción |
