# CodLab N°10 — Backend Spring Boot Completo + Integración Android
**Curso:** Desarrollo de Sistemas Móviles | **Semana:** 11  
**Evaluación:** Individual

---

## Objetivos

- Construir una API REST completa con Spring Boot 3.3+ + Kotlin.
- Implementar patrón DTO + Mapper + Service + Controller.
- Agregar validación de datos y manejo de errores.
- Consumir la API completa desde Android con CRUD completo.
- Integrar Firebase JWT con Spring Security para proteger los endpoints.

## Prerrequisitos

- CodLab N°9 completado (base de Spring Boot funcionando).
- App Android con Retrofit configurado.
- Firebase proyecto activo.

---

## Paso 1: Mejorar la estructura del proyecto

Reestructura tu proyecto Spring Boot con esta organización:

```
src/main/kotlin/com/tuapellido/dsmbackend/
├── config/
│   ├── SecurityConfig.kt
│   └── DataInitializer.kt
├── controller/
│   └── ProductoController.kt
├── service/
│   └── ProductoService.kt          ← lógica de negocio separada del controller
├── repository/
│   └── ProductoRepository.kt
├── entity/
│   └── Producto.kt
├── dto/
│   ├── ProductoDto.kt
│   ├── CreateProductoRequest.kt
│   └── UpdateProductoRequest.kt
├── mapper/
│   └── ProductoMapper.kt           ← conversión Entity ↔ DTO
└── exception/
    ├── NotFoundException.kt
    └── GlobalExceptionHandler.kt
```

---

## Paso 2: Service layer

```kotlin
// service/ProductoService.kt
@Service
class ProductoService(private val repository: ProductoRepository) {

    fun getAll(q: String? = null, categoria: String? = null): List<ProductoDto> {
        val productos = when {
            q != null -> repository.findByNombreContainingIgnoreCase(q)
            categoria != null -> repository.findByCategoriaOrderByNombreAsc(categoria)
            else -> repository.findAll()
        }
        return productos.map { it.toDto() }
    }

    fun getById(id: Long): ProductoDto =
        repository.findById(id)
            .map { it.toDto() }
            .orElseThrow { NotFoundException("Producto con id $id no encontrado") }

    fun create(request: CreateProductoRequest): ProductoDto =
        repository.save(
            Producto(nombre = request.nombre, precio = request.precio, categoria = request.categoria)
        ).toDto()

    fun update(id: Long, request: UpdateProductoRequest): ProductoDto {
        val existente = repository.findById(id)
            .orElseThrow { NotFoundException("Producto con id $id no encontrado") }
        return repository.save(existente.copy(
            nombre = request.nombre ?: existente.nombre,
            precio = request.precio ?: existente.precio,
            categoria = request.categoria ?: existente.categoria,
            disponible = request.disponible ?: existente.disponible
        )).toDto()
    }

    fun delete(id: Long) {
        if (!repository.existsById(id)) throw NotFoundException("Producto con id $id no encontrado")
        repository.deleteById(id)
    }

    private fun Producto.toDto() = ProductoDto(id, nombre, precio, categoria, disponible)
}
```

---

## Paso 3: DTOs y manejo de errores

```kotlin
// dto/ProductoDto.kt
data class ProductoDto(
    val id: Long,
    val nombre: String,
    val precio: Double,
    val categoria: String,
    val disponible: Boolean
)

// dto/CreateProductoRequest.kt
data class CreateProductoRequest(
    @field:NotBlank(message = "El nombre es requerido")
    @field:Size(min = 2, max = 200)
    val nombre: String,
    @field:Positive(message = "El precio debe ser mayor a 0")
    val precio: Double,
    @field:NotBlank(message = "La categoría es requerida")
    val categoria: String
)

// dto/UpdateProductoRequest.kt
data class UpdateProductoRequest(
    @field:Size(min = 2, max = 200)
    val nombre: String? = null,
    @field:Positive
    val precio: Double? = null,
    val categoria: String? = null,
    val disponible: Boolean? = null
)

// exception/NotFoundException.kt
class NotFoundException(message: String) : RuntimeException(message)

// exception/GlobalExceptionHandler.kt
@RestControllerAdvice
class GlobalExceptionHandler {

    data class ApiError(val error: String, val message: String, val timestamp: Long = System.currentTimeMillis())

    @ExceptionHandler(NotFoundException::class)
    @ResponseStatus(HttpStatus.NOT_FOUND)
    fun handleNotFound(ex: NotFoundException) = ApiError("NOT_FOUND", ex.message ?: "Recurso no encontrado")

    @ExceptionHandler(MethodArgumentNotValidException::class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    fun handleValidation(ex: MethodArgumentNotValidException): Map<String, Any> = mapOf(
        "error" to "VALIDATION_ERROR",
        "fields" to ex.bindingResult.fieldErrors.associate { it.field to it.defaultMessage }
    )

    @ExceptionHandler(Exception::class)
    @ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR)
    fun handleGeneric(ex: Exception) = ApiError("INTERNAL_ERROR", "Error interno del servidor")
}
```

---

## Paso 4: Controller limpio

```kotlin
// controller/ProductoController.kt
@RestController
@RequestMapping("/api/v1/productos")
@CrossOrigin(origins = ["*"])
class ProductoController(private val service: ProductoService) {

    @GetMapping
    fun getAll(
        @RequestParam(required = false) q: String?,
        @RequestParam(required = false) categoria: String?
    ) = service.getAll(q, categoria)

    @GetMapping("/{id}")
    fun getById(@PathVariable id: Long) = service.getById(id)

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    fun create(@Valid @RequestBody request: CreateProductoRequest) = service.create(request)

    @PatchMapping("/{id}")
    fun update(@PathVariable id: Long, @Valid @RequestBody request: UpdateProductoRequest) =
        service.update(id, request)

    @DeleteMapping("/{id}")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    fun delete(@PathVariable id: Long) = service.delete(id)
}
```

---

## Paso 5: Proteger con Firebase JWT

```kotlin
// config/SecurityConfig.kt — verificar tokens de Firebase Auth
@Configuration
@EnableWebSecurity
class SecurityConfig {

    @Bean
    fun securityFilterChain(http: HttpSecurity): SecurityFilterChain {
        http
            .csrf { it.disable() }
            .cors { } // usa el bean CorsConfigurationSource
            .authorizeHttpRequests { auth ->
                auth.requestMatchers(HttpMethod.GET, "/api/v1/productos/**").permitAll()
                auth.requestMatchers("/swagger-ui/**", "/api-docs/**", "/h2-console/**").permitAll()
                auth.anyRequest().authenticated()
            }
            .addFilterBefore(FirebaseAuthFilter(), UsernamePasswordAuthenticationFilter::class.java)
        return http.build()
    }
}

// FirebaseAuthFilter.kt — validar el token JWT de Firebase en cada petición
@Component
class FirebaseAuthFilter : OncePerRequestFilter() {
    override fun doFilterInternal(request: HttpServletRequest, response: HttpServletResponse, chain: FilterChain) {
        val header = request.getHeader("Authorization")
        if (header != null && header.startsWith("Bearer ")) {
            try {
                val token = header.substring(7)
                val decodedToken = FirebaseAuth.getInstance().verifyIdToken(token)
                val authentication = UsernamePasswordAuthenticationToken(
                    decodedToken.uid, null, emptyList()
                )
                SecurityContextHolder.getContext().authentication = authentication
            } catch (e: Exception) {
                response.sendError(HttpServletResponse.SC_UNAUTHORIZED, "Token inválido")
                return
            }
        }
        chain.doFilter(request, response)
    }
}
```

---

## Paso 6: Android — CRUD completo con ViewModel

```kotlin
// ui/productos/ProductosViewModel.kt
@HiltViewModel
class ProductosCrudViewModel @Inject constructor(
    private val api: ProductosLocalApi
) : ViewModel() {

    data class UiState(
        val productos: List<ProductoApiDto> = emptyList(),
        val isLoading: Boolean = false,
        val error: String? = null,
        val successMessage: String? = null
    )

    private val _uiState = MutableStateFlow(UiState())
    val uiState: StateFlow<UiState> = _uiState.asStateFlow()

    init { cargar() }

    fun cargar() {
        viewModelScope.launch {
            _uiState.update { it.copy(isLoading = true) }
            runCatching { api.getProductos() }
                .onSuccess { lista -> _uiState.update { it.copy(productos = lista, isLoading = false) } }
                .onFailure { e -> _uiState.update { it.copy(error = e.message, isLoading = false) } }
        }
    }

    fun crear(nombre: String, precio: Double, categoria: String) {
        viewModelScope.launch {
            runCatching { api.createProducto(CreateProductoApiRequest(nombre, precio, categoria)) }
                .onSuccess { 
                    cargar()
                    _uiState.update { it.copy(successMessage = "Producto creado exitosamente") }
                }
                .onFailure { e -> _uiState.update { it.copy(error = e.message) } }
        }
    }

    fun eliminar(id: Long) {
        viewModelScope.launch {
            runCatching { api.deleteProducto(id) }
                .onSuccess { cargar() }
                .onFailure { e -> _uiState.update { it.copy(error = e.message) } }
        }
    }

    fun dismissError() = _uiState.update { it.copy(error = null) }
    fun dismissSuccess() = _uiState.update { it.copy(successMessage = null) }
}
```

---

## Entregables

1. Backend con Service layer separado del Controller.
2. `GET /api/v1/productos?q=laptop` retorna resultados filtrados.
3. `POST /api/v1/productos` con datos inválidos retorna errores por campo (JSON).
4. `PATCH /api/v1/productos/{id}` actualiza solo los campos enviados.
5. App Android hace CRUD completo: listar + crear + eliminar.
6. App Android envía el token JWT de Firebase en el header `Authorization`.

## Criterios de Evaluación

| Criterio | Puntaje |
|---|---|
| Service layer con lógica separada del Controller | 2 pts |
| Errores de validación retornan mensajes por campo | 1 pt |
| PATCH implementado (actualización parcial) | 1 pt |
| App Android hace CRUD completo (crear + listar + eliminar) | 3 pts |
| Token JWT de Firebase enviado en las peticiones protegidas | 2 pts |
| Manejo de errores en Android (sin crashes por error 400/500) | 1 pt |
| **Total** | **10 pts** |
