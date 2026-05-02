# Semana 11 — Frameworks Backend para Apps Móviles
**Curso:** Desarrollo de Sistemas Móviles | **Unidad:** U3  
**Sesión:** Presencial | **Duración:** 2 horas

---

## Objetivos de la Sesión

- Comparar los frameworks backend más relevantes en 2026: Spring Boot, Ktor, Quarkus, FastAPI.
- Entender la arquitectura de microservicios y cuándo aplicarla.
- Implementar un backend con Spring Boot 3.3+ usando Virtual Threads.
- Conocer los patrones CQRS y Event-Driven Architecture.
- Diseñar una API con spec-first approach usando OpenAPI 3.1.

---

## 1. Panorama de Frameworks Backend en 2026 *(20 min)*

### 1.1 Comparativa de frameworks

| Framework | Lenguaje | Startup Time | RAM en Docker | Mejor para |
|---|---|---|---|---|
| **Spring Boot 3.3+** | Java/Kotlin | ~2s (JVM) / ~50ms (GraalVM) | 200MB / 50MB native | Enterprise, ecosistema maduro |
| **Quarkus 3.8+** | Java/Kotlin | ~300ms (JVM) / ~50ms native | 120MB / 30MB native | Cloud-native, Kubernetes-first |
| **Ktor 2.3+** | Kotlin | ~200ms | 80MB | Equipos Kotlin, APIs ligeras |
| **FastAPI 0.110+** | Python | ~500ms | 100MB | Equipos Python, IA/ML backends |
| **Go Echo 4.12+** | Go | ~10ms | 15MB | Alta concurrencia, microservicios |

Fuente de benchmarks: [TechEmpower Framework Benchmarks](https://www.techempower.com/benchmarks/)

> **Estadística 2024:** Spring Boot es el framework web más usado a nivel global, según el [Stack Overflow Developer Survey 2024](https://survey.stackoverflow.co/2024/technology#most-popular-technologies-webframe). Quarkus crece rápidamente en entornos cloud-native.

### 1.2 ¿Qué framework elegir?

```
¿El equipo sabe Java/Kotlin y necesita integración empresarial?
    → Spring Boot 3.3+

¿El proyecto corre en Kubernetes con requisito de startup < 1 segundo?
    → Quarkus 3.8+ con GraalVM native image

¿El equipo es 100% Kotlin y quiere control total sobre la configuración?
    → Ktor 2.3+

¿El equipo es Python, el backend tiene mucha IA/ML?
    → FastAPI 0.110+

¿Necesitas máxima concurrencia con mínima RAM?
    → Go Echo
```

---

## 2. Spring Boot 3.3+ — El Framework Enterprise *(30 min)*

### 2.1 Novedades Spring Boot 3.3+ en 2026

**Virtual Threads (Project Loom, Java 21+):**
```properties
# application.properties — habilitar virtual threads
spring.threads.virtual.enabled=true
```

Con Virtual Threads, puedes escribir código bloqueante (que parece síncrono) y Spring lo ejecuta de forma no bloqueante automáticamente. El número de hilos virtuales es millones, vs ~200 hilos del pool tradicional.

```kotlin
// Con Virtual Threads, este código "bloqueante" escala como si fuera reactivo
@GetMapping("/lento")
fun endpointLento(): List<Producto> {
    Thread.sleep(1000)  // virtual thread se suspende, no bloquea el hilo del pool
    return repository.findAll()  // query lenta — OK con virtual threads
}
```

**GraalVM Native Image:**
```bash
# Compilar a binario nativo — app inicia en ~50ms
./gradlew nativeCompile
./build/native/nativeCompile/dsm-backend  # ~50ms startup
```

**Spring Security 6.2:**
```kotlin
@Configuration
@EnableWebSecurity
class SecurityConfig {

    @Bean
    fun securityFilterChain(http: HttpSecurity): SecurityFilterChain {
        http
            .csrf { it.disable() }  // APIs REST sin sesión no necesitan CSRF
            .authorizeHttpRequests {
                it.requestMatchers("/api/public/**").permitAll()
                it.requestMatchers("/api/v1/**").authenticated()
                it.requestMatchers("/admin/**").hasRole("ADMIN")
            }
            .oauth2ResourceServer { it.jwt { } }  // JWT de Firebase Auth o Auth0
        return http.build()
    }
}
```

### 2.2 Estructura recomendada para proyectos Spring Boot + Kotlin

```
src/main/kotlin/
├── config/           → SecurityConfig, CorsConfig, SwaggerConfig
├── controller/       → RestController — solo routing y mapeo HTTP
├── service/          → Lógica de negocio (equivale a UseCase en Android)
├── repository/       → Spring Data JPA repositories
├── entity/           → @Entity clases (tabla de BD)
├── dto/              → Data Transfer Objects (request/response)
├── mapper/           → Conversiones Entity ↔ DTO
└── exception/        → Excepciones custom + @ControllerAdvice
```

---

## 3. Ktor 2.3+ — El Framework Kotlin-Native *(20 min)*

Ktor es desarrollado por JetBrains. Usa coroutines nativas y configuración DSL:

```kotlin
fun main() {
    embeddedServer(Netty, port = 8080) {
        // Plugins (equivalentes a los starters de Spring Boot)
        install(ContentNegotiation) { json() }
        install(CallLogging)
        install(Authentication) {
            jwt("auth-jwt") {
                // configurar JWT
            }
        }
        install(CORS) {
            anyHost()
            allowHeader(HttpHeaders.ContentType)
        }

        routing {
            route("/api/v1") {
                get("/productos") {
                    val productos = productoService.getAll()
                    call.respond(productos)
                }
                post("/productos") {
                    val request = call.receive<CreateProductoRequest>()
                    val producto = productoService.create(request)
                    call.respond(HttpStatusCode.Created, producto)
                }
                authenticate("auth-jwt") {
                    delete("/productos/{id}") {
                        val id = call.parameters["id"]?.toIntOrNull()
                            ?: return@delete call.respond(HttpStatusCode.BadRequest)
                        productoService.delete(id)
                        call.respond(HttpStatusCode.NoContent)
                    }
                }
            }
        }
    }.start(wait = true)
}
```

---

## 4. Arquitectura de Microservicios *(20 min)*

### 4.1 Monolítico vs Microservicios

```
MONOLÍTICO:
┌─────────────────────────────────────────┐
│           Backend Único                 │
│  Auth + Productos + Órdenes + Usuarios  │
│         1 base de datos                 │
└─────────────────────────────────────────┘
+ Simple de desarrollar inicialmente
+ Fácil de debuggear
- Escala todo o nada
- Un bug puede tumbar toda la app

MICROSERVICIOS:
┌──────────┐  ┌──────────┐  ┌──────────┐
│  Auth    │  │Productos │  │ Órdenes  │
│ Service  │  │ Service  │  │ Service  │
└──────────┘  └──────────┘  └──────────┘
     │               │            │
  PostgreSQL      PostgreSQL   PostgreSQL
(base de datos separada por servicio)
+ Escala independientemente
+ Fallos aislados
- Complejidad de red y comunicación
- Difícil de testear localmente
```

### 4.2 API Gateway Pattern

El cliente móvil habla con un solo punto de entrada:

```
App Android
     ↓ HTTPS
  API Gateway  (autenticación, rate limiting, routing)
  ┌──────┬────────┬────────┐
  ↓      ↓        ↓        ↓
Auth  Productos  Órdenes  Usuarios
 Service Service  Service  Service
```

### 4.3 Event-Driven Architecture con Kafka

En lugar de llamadas síncronas entre servicios, los servicios publican y consumen eventos:

```kotlin
// Servicio de Órdenes publica un evento al crearse una orden
kafkaTemplate.send("orden-creada", OrdenCreadaEvent(
    ordenId = orden.id,
    usuarioId = orden.usuarioId,
    productos = orden.productos
))

// Servicio de Inventario escucha el evento y descuenta stock
@KafkaListener(topics = ["orden-creada"])
fun handleOrdenCreada(event: OrdenCreadaEvent) {
    event.productos.forEach { item ->
        inventarioService.descontarStock(item.productoId, item.cantidad)
    }
}
```

---

## 5. OpenAPI 3.1 — API-First Design *(15 min)*

El enfoque **API-First** define el contrato de la API antes de implementarla:

```yaml
# openapi.yaml — escrito primero, luego se genera el código
openapi: 3.1.0
info:
  title: DSM Products API
  version: 1.0.0

paths:
  /api/v1/productos:
    get:
      summary: Obtener lista de productos
      parameters:
        - name: q
          in: query
          schema: {type: string}
          description: Buscar por nombre
      responses:
        '200':
          description: Lista de productos
          content:
            application/json:
              schema:
                type: array
                items: {$ref: '#/components/schemas/Producto'}

components:
  schemas:
    Producto:
      type: object
      required: [id, nombre, precio, categoria]
      properties:
        id: {type: integer, format: int64}
        nombre: {type: string, maxLength: 200}
        precio: {type: number, minimum: 0}
        categoria: {type: string}
        disponible: {type: boolean, default: true}
```

**Beneficio para apps móviles:** el equipo Android puede generar el cliente Retrofit automáticamente desde el YAML antes de que el backend esté implementado — desarrollan en paralelo.

---

## 6. Resumen de la Sesión

| Concepto | Clave a recordar |
|---|---|
| Spring Boot 3.3+ | Virtual Threads en Java 21: escala bloqueante como reactivo. GraalVM native: 50ms startup. |
| Quarkus | Cloud-native desde el diseño. Dev Services arranca PostgreSQL/Redis automáticamente. |
| Ktor | DSL de Kotlin, coroutines nativas, máximo control. Menor magia que Spring. |
| FastAPI | Python async, documentación automática en /docs, Pydantic v2 para validación. |
| Microservicios | Escala independiente + fallos aislados. Añade complejidad operacional. |
| Event-Driven | Servicios desacoplados vía Kafka/Redis. Publicar eventos, no llamar directamente. |
| OpenAPI 3.1 | Contrato primero → generación de código. Frontend y backend desarrollan en paralelo. |

---

## Tarea Previa al CodLab

1. Asegúrate de tener IntelliJ IDEA y JDK 21 instalados.
2. Crea un proyecto en [start.spring.io](https://start.spring.io) con: Spring Web, Spring Data JPA, PostgreSQL Driver, Validation.
3. Configura PostgreSQL local (con Docker: `docker run -e POSTGRES_PASSWORD=postgres -p 5432:5432 postgres:16-alpine`).

---

## Recursos Adicionales

| Recurso | Enlace | Para qué |
|---|---|---|
| Spring Boot Docs | [spring.io/projects/spring-boot](https://spring.io/projects/spring-boot) | Referencia oficial |
| Ktor Docs | [ktor.io/docs](https://ktor.io/docs/) | Framework Kotlin-native |
| Quarkus Guides | [quarkus.io/guides](https://quarkus.io/guides/) | Cloud-native Java |
| FastAPI Docs | [fastapi.tiangolo.com](https://fastapi.tiangolo.com/) | Framework Python |
| TechEmpower Benchmarks | [techempower.com/benchmarks](https://www.techempower.com/benchmarks/) | Comparativa de rendimiento |
| Stack Overflow Survey 2024 | [survey.stackoverflow.co/2024](https://survey.stackoverflow.co/2024/) | Popularidad de frameworks |
