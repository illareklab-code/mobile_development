# CodLab N°9 — Consumo de APIs REST con Backend Spring Boot
**Curso:** Desarrollo de Sistemas Móviles | **Semana:** 10  
**Evaluación:** Individual

---

## Objetivos

- Crear un backend Spring Boot 3.3+ con Kotlin y Spring Data JPA.
- Implementar CRUD completo con endpoints REST documentados.
- Consumir la API desde la app Android con Retrofit.
- Usar H2 in-memory para el entorno de laboratorio.
- Explorar la documentación OpenAPI generada automáticamente.

## Prerrequisitos

- IntelliJ IDEA Community (gratuita) o Ultimate.
- JDK 21 instalado (`java -version` debe mostrar 21+).
- Postman o Bruno instalado para probar APIs.

---

## Parte 1: Backend Spring Boot (60 min)

### Paso 1: Crear el proyecto Spring Boot

1. Ve a [start.spring.io](https://start.spring.io).
2. Configura:
   - **Project:** Gradle - Kotlin
   - **Language:** Kotlin
   - **Spring Boot:** 3.3.x
   - **Group:** `com.tuapellido`
   - **Artifact:** `dsm-backend`
   - **Dependencies:** Spring Web, Spring Data JPA, H2 Database, Validation, SpringDoc OpenAPI
3. Descarga y abre en IntelliJ IDEA.

### Paso 2: Dependencias adicionales

```kotlin
// build.gradle.kts — agregar SpringDoc para documentación automática
dependencies {
    implementation("org.springdoc:springdoc-openapi-starter-webmvc-ui:2.5.0")
}
```

### Paso 3: Modelo de datos

```kotlin
// entity/Producto.kt
@Entity
@Table(name = "productos")
data class Producto(
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    val id: Long = 0,

    @field:NotBlank(message = "El nombre es obligatorio")
    @field:Size(min = 2, max = 200, message = "El nombre debe tener entre 2 y 200 caracteres")
    @Column(nullable = false)
    val nombre: String,

    @field:Positive(message = "El precio debe ser positivo")
    @Column(nullable = false)
    val precio: Double,

    @field:NotBlank
    @Column(nullable = false)
    val categoria: String,

    val disponible: Boolean = true
)

// dto/ProductoRequest.kt
data class ProductoRequest(
    @field:NotBlank val nombre: String,
    @field:Positive val precio: Double,
    @field:NotBlank val categoria: String
)
```

### Paso 4: Repository con queries personalizadas

```kotlin
// repository/ProductoRepository.kt
interface ProductoRepository : JpaRepository<Producto, Long> {
    fun findByDisponibleTrue(): List<Producto>
    fun findByCategoriaOrderByNombreAsc(categoria: String): List<Producto>
    fun findByNombreContainingIgnoreCase(nombre: String): List<Producto>

    @Query("SELECT p FROM Producto p WHERE p.precio BETWEEN :min AND :max")
    fun findByRangoPrecio(@Param("min") min: Double, @Param("max") max: Double): List<Producto>
}
```

### Paso 5: Controller REST

```kotlin
// controller/ProductoController.kt
@RestController
@RequestMapping("/api/v1/productos")
@CrossOrigin(origins = ["*"])  // para el emulador Android
class ProductoController(private val repository: ProductoRepository) {

    @GetMapping
    fun getAll(@RequestParam(required = false) q: String?): List<Producto> =
        if (q != null) repository.findByNombreContainingIgnoreCase(q)
        else repository.findAll()

    @GetMapping("/{id}")
    fun getById(@PathVariable id: Long): ResponseEntity<Producto> =
        repository.findById(id)
            .map { ResponseEntity.ok(it) }
            .orElse(ResponseEntity.notFound().build())

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    fun create(@Valid @RequestBody request: ProductoRequest): Producto =
        repository.save(Producto(
            nombre = request.nombre,
            precio = request.precio,
            categoria = request.categoria
        ))

    @PutMapping("/{id}")
    fun update(@PathVariable id: Long, @Valid @RequestBody request: ProductoRequest): ResponseEntity<Producto> {
        if (!repository.existsById(id)) return ResponseEntity.notFound().build()
        return ResponseEntity.ok(repository.save(Producto(
            id = id,
            nombre = request.nombre,
            precio = request.precio,
            categoria = request.categoria
        )))
    }

    @DeleteMapping("/{id}")
    fun delete(@PathVariable id: Long): ResponseEntity<Void> {
        if (!repository.existsById(id)) return ResponseEntity.notFound().build()
        repository.deleteById(id)
        return ResponseEntity.noContent().build()
    }
}

// Manejo global de errores
@RestControllerAdvice
class GlobalExceptionHandler {

    @ExceptionHandler(MethodArgumentNotValidException::class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    fun handleValidation(ex: MethodArgumentNotValidException): Map<String, String?> =
        ex.bindingResult.fieldErrors.associate { it.field to it.defaultMessage }
}
```

### Paso 6: Configuración y datos iniciales

```properties
# src/main/resources/application.properties
spring.h2.console.enabled=true
spring.h2.console.path=/h2-console
spring.datasource.url=jdbc:h2:mem:testdb
spring.jpa.show-sql=true
spring.jpa.hibernate.ddl-auto=create-drop

# OpenAPI / Swagger UI
springdoc.swagger-ui.path=/swagger-ui.html
springdoc.api-docs.path=/api-docs
```

```kotlin
// DataInitializer.kt — datos de prueba al iniciar
@Component
class DataInitializer(private val repository: ProductoRepository) : ApplicationRunner {
    override fun run(args: ApplicationArguments) {
        if (repository.count() == 0L) {
            repository.saveAll(listOf(
                Producto(nombre = "Laptop Dell XPS 15", precio = 5200.0, categoria = "Tecnología"),
                Producto(nombre = "Auriculares Sony WH-1000XM5", precio = 890.0, categoria = "Tecnología"),
                Producto(nombre = "Zapatillas Nike Air Max", precio = 450.0, categoria = "Ropa"),
                Producto(nombre = "Cafetera Nespresso", precio = 320.0, categoria = "Hogar"),
                Producto(nombre = "Polo algodón pima", precio = 85.0, categoria = "Ropa")
            ))
        }
    }
}
```

### Paso 7: Probar con Postman

1. Inicia el proyecto: `./gradlew bootRun` → espera `Started DsmBackendApplication`.
2. Abre Swagger UI: [http://localhost:8080/swagger-ui.html](http://localhost:8080/swagger-ui.html).
3. Prueba cada endpoint desde Swagger UI o Postman:
   - `GET /api/v1/productos` → debe retornar lista de 5 productos.
   - `POST /api/v1/productos` con body `{"nombre":"Test","precio":100,"categoria":"Test"}` → 201 Created.
   - `GET /api/v1/productos?q=sony` → debe filtrar por nombre.
   - `DELETE /api/v1/productos/1` → 204 No Content.

---

## Parte 2: Consumir la API desde Android (40 min)

### Paso 8: Actualizar el ApiService en Android

```kotlin
// data/remote/api/ProductosLocalApi.kt
interface ProductosLocalApi {
    // 10.0.2.2 = localhost desde el emulador Android
    // Si usas dispositivo físico: usar la IP de tu PC en la red local (ej: 192.168.1.100)

    @GET("api/v1/productos")
    suspend fun getProductos(@Query("q") query: String? = null): List<ProductoApiDto>

    @GET("api/v1/productos/{id}")
    suspend fun getProducto(@Path("id") id: Long): ProductoApiDto

    @POST("api/v1/productos")
    suspend fun createProducto(@Body request: CreateProductoApiRequest): ProductoApiDto

    @PUT("api/v1/productos/{id}")
    suspend fun updateProducto(
        @Path("id") id: Long,
        @Body request: CreateProductoApiRequest
    ): ProductoApiDto

    @DELETE("api/v1/productos/{id}")
    suspend fun deleteProducto(@Path("id") id: Long)
}

@Serializable
data class ProductoApiDto(
    val id: Long,
    val nombre: String,
    val precio: Double,
    val categoria: String,
    val disponible: Boolean
)

@Serializable
data class CreateProductoApiRequest(
    val nombre: String,
    val precio: Double,
    val categoria: String
)
```

### Paso 9: Actualizar NetworkModule para apuntar al backend local

```kotlin
// di/NetworkModule.kt
@Provides
@Singleton
fun provideRetrofit(okHttpClient: OkHttpClient, json: Json): Retrofit =
    Retrofit.Builder()
        .baseUrl("http://10.0.2.2:8080/")  // localhost desde emulador
        .client(okHttpClient)
        .addConverterFactory(json.asConverterFactory("application/json".toMediaType()))
        .build()
```

En `network_security_config.xml` (crear en `res/xml/`):
```xml
<?xml version="1.0" encoding="utf-8"?>
<network-security-config>
    <domain-config cleartextTrafficPermitted="true">
        <domain includeSubdomains="false">10.0.2.2</domain>
    </domain-config>
</network-security-config>
```

Y en `AndroidManifest.xml`: `android:networkSecurityConfig="@xml/network_security_config"`.

### Paso 10: Probar la integración

1. Asegúrate de que el backend Spring Boot está corriendo en tu PC.
2. Inicia el emulador Android.
3. Corre la app Android.
4. La lista de productos debe mostrar los datos del backend Spring Boot.
5. Verifica los logs en el OkHttp interceptor (Logcat tag `OkHttp`).

---

## Entregables

1. Backend Spring Boot corriendo con 5 endpoints CRUD.
2. Swagger UI accesible mostrando todos los endpoints.
3. App Android mostrando la lista de productos del backend local.
4. Capturas de pantalla de: Swagger UI + app mostrando los datos.

## Criterios de Evaluación

| Criterio | Puntaje |
|---|---|
| Backend con 5 endpoints CRUD funcionando | 3 pts |
| Validación con `@Valid` retorna errores descriptivos | 1 pt |
| Swagger UI muestra documentación automática | 1 pt |
| App Android consume el backend local | 3 pts |
| OkHttp logging muestra las peticiones al backend | 1 pt |
| Sin claves hardcodeadas en el código | 1 pt |
| **Total** | **10 pts** |
