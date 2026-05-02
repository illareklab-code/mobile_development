# Semana 10 — ORM/DAO para Backend: Exposed, JPA, Jooq
**Curso:** Desarrollo de Sistemas Móviles | **Unidad:** U3  
**Sesión:** Presencial | **Duración:** 2 horas

---

## Objetivos de la Sesión

- Comparar los frameworks ORM/DAO disponibles para backends Kotlin/Java en 2026.
- Implementar queries type-safe con Exposed DSL y Exposed DAO.
- Entender Jakarta Persistence API (JPA 3.2) e Hibernate 6.4+.
- Aplicar Flyway 10 para migraciones versionadas de base de datos.
- Introducir Testcontainers para testing con bases de datos reales.

---

## 1. ¿Qué es un ORM? *(15 min)*

Un **ORM** (Object-Relational Mapping) mapea tablas de base de datos a clases del lenguaje y viceversa, eliminando la necesidad de escribir SQL manualmente en la mayoría de casos.

**Sin ORM:**
```kotlin
val sql = "SELECT id, nombre, precio FROM productos WHERE disponible = true ORDER BY nombre"
val stmt = connection.prepareStatement(sql)
val rs = stmt.executeQuery()
val productos = mutableListOf<Producto>()
while (rs.next()) {
    productos.add(Producto(rs.getInt("id"), rs.getString("nombre"), rs.getDouble("precio")))
}
```

**Con ORM (Room o Exposed):**
```kotlin
// Room en Android
val productos = productoDao.getAll().first()

// Exposed en Backend
val productos = Productos.selectAll().where { Productos.disponible eq true }.map { it.toProducto() }
```

### Comparativa de frameworks ORM/DAO 2026

| Framework | Lenguaje | Tipo | Type-Safe | Cuándo usarlo |
|---|---|---|---|---|
| **Room** | Kotlin | ORM Android | ✅ En queries | Apps Android locales |
| **Exposed DSL** | Kotlin | Query builder | ✅ Total | Backends Kotlin, proyectos medianos |
| **Exposed DAO** | Kotlin | Active Record | ✅ Total | CRUD simple, prototipado |
| **Spring Data JPA** | Java/Kotlin | ORM estándar | Parcial | Apps Spring Boot enterprise |
| **Hibernate 6.4+** | Java/Kotlin | ORM full-featured | Parcial | Sistemas legados, JPA requirement |
| **Jooq 3.19+** | Java/Kotlin | Query builder | ✅ Total | SQL complejo, esquemas existentes |
| **R2DBC** | Java/Kotlin | Reactivo | Parcial | Backends reactivos, alta concurrencia |

---

## 2. Exposed — El ORM de JetBrains *(35 min)*

Exposed es el framework de acceso a base de datos de JetBrains para Kotlin. Tiene dos APIs:

### 2.1 Exposed DSL (recomendado)

```kotlin
// build.gradle.kts (backend Spring Boot / Ktor)
implementation("org.jetbrains.exposed:exposed-core:0.54.0")
implementation("org.jetbrains.exposed:exposed-dao:0.54.0")
implementation("org.jetbrains.exposed:exposed-jdbc:0.54.0")
implementation("org.jetbrains.exposed:exposed-kotlin-datetime:0.54.0")
implementation("org.postgresql:postgresql:42.7.3")

// Definir tabla como objeto Kotlin
object Productos : Table("productos") {
    val id = integer("id").autoIncrement()
    val nombre = varchar("nombre", 200)
    val precio = decimal("precio", precision = 10, scale = 2)
    val categoria = varchar("categoria", 50)
    val disponible = bool("disponible").default(true)
    val creadoEn = datetime("creado_en").defaultExpression(CurrentDateTime)
    override val primaryKey = PrimaryKey(id)
}

// Consultas DSL
object ProductosRepository {

    fun getAll(disponible: Boolean? = null): List<ProductoDto> = transaction {
        Productos.selectAll()
            .apply { if (disponible != null) where { Productos.disponible eq disponible } }
            .orderBy(Productos.nombre to SortOrder.ASC)
            .map { it.toDto() }
    }

    fun getById(id: Int): ProductoDto? = transaction {
        Productos.selectAll()
            .where { Productos.id eq id }
            .singleOrNull()
            ?.toDto()
    }

    fun crear(request: CreateProductoRequest): ProductoDto = transaction {
        val id = Productos.insertAndGetId {
            it[nombre] = request.nombre
            it[precio] = request.precio.toBigDecimal()
            it[categoria] = request.categoria
            it[disponible] = true
        }
        getById(id.value)!!
    }

    fun actualizar(id: Int, request: UpdateProductoRequest): ProductoDto? = transaction {
        val updated = Productos.update({ Productos.id eq id }) {
            request.nombre?.let { nombre -> it[Productos.nombre] = nombre }
            request.precio?.let { precio -> it[Productos.precio] = precio.toBigDecimal() }
        }
        if (updated > 0) getById(id) else null
    }

    fun eliminar(id: Int): Boolean = transaction {
        Productos.deleteWhere { Productos.id eq id } > 0
    }

    fun buscar(query: String): List<ProductoDto> = transaction {
        Productos.selectAll()
            .where { Productos.nombre like "%$query%" }
            .map { it.toDto() }
    }

    // Mapper de ResultRow a DTO
    private fun ResultRow.toDto() = ProductoDto(
        id = this[Productos.id],
        nombre = this[Productos.nombre],
        precio = this[Productos.precio].toDouble(),
        categoria = this[Productos.categoria],
        disponible = this[Productos.disponible]
    )
}
```

### 2.2 Configuración de la conexión

```kotlin
// DatabaseConfig.kt
object DatabaseConfig {
    fun init(
        url: String = System.getenv("DATABASE_URL") ?: "jdbc:postgresql://localhost:5432/mi_db",
        driver: String = "org.postgresql.Driver",
        user: String = System.getenv("DB_USER") ?: "postgres",
        password: String = System.getenv("DB_PASSWORD") ?: ""
    ) {
        Database.connect(
            url = url,
            driver = driver,
            user = user,
            password = password
        )
        // Crear tablas si no existen (solo en desarrollo)
        transaction {
            SchemaUtils.create(Productos)
        }
    }
}
```

---

## 3. Jakarta Persistence API (JPA) con Hibernate 6.4+ *(20 min)*

JPA es el estándar de la industria para ORM en Java/Kotlin. Spring Boot usa JPA + Hibernate por defecto.

```kotlin
// Entidad JPA en Kotlin
@Entity
@Table(name = "productos")
data class ProductoEntity(
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    val id: Long = 0,

    @Column(nullable = false, length = 200)
    val nombre: String,

    @Column(nullable = false, precision = 10, scale = 2)
    val precio: BigDecimal,

    @Column(nullable = false, length = 50)
    val categoria: String,

    val disponible: Boolean = true,

    @CreationTimestamp
    val creadoEn: LocalDateTime = LocalDateTime.now()
)

// Spring Data JPA Repository — queries sin código SQL
interface ProductoRepository : JpaRepository<ProductoEntity, Long> {
    // Spring genera la query automáticamente desde el nombre del método
    fun findByDisponibleTrue(): List<ProductoEntity>
    fun findByCategoriaOrderByNombreAsc(categoria: String): List<ProductoEntity>
    fun findByNombreContainingIgnoreCase(query: String): List<ProductoEntity>

    // Query personalizada cuando el nombre no alcanza
    @Query("SELECT p FROM ProductoEntity p WHERE p.precio BETWEEN :min AND :max")
    fun findByRangoPrecio(
        @Param("min") precioMin: BigDecimal,
        @Param("max") precioMax: BigDecimal
    ): List<ProductoEntity>
}
```

---

## 4. Flyway 10 — Migraciones Versionadas *(15 min)*

Flyway controla los cambios del esquema de base de datos a lo largo del tiempo.

```
src/main/resources/db/migration/
├── V1__create_products_table.sql
├── V2__add_stock_column.sql
├── V3__create_categories_table.sql
└── V4__add_foreign_key_category.sql
```

```sql
-- V1__create_products_table.sql
CREATE TABLE productos (
    id SERIAL PRIMARY KEY,
    nombre VARCHAR(200) NOT NULL,
    precio DECIMAL(10,2) NOT NULL,
    categoria VARCHAR(50) NOT NULL,
    disponible BOOLEAN NOT NULL DEFAULT true,
    creado_en TIMESTAMP NOT NULL DEFAULT NOW()
);

-- V2__add_stock_column.sql
ALTER TABLE productos ADD COLUMN stock INTEGER NOT NULL DEFAULT 0;
CREATE INDEX idx_productos_categoria ON productos(categoria);
```

```kotlin
// Spring Boot — Flyway se configura automáticamente con estas propiedades:
// application.properties:
// spring.flyway.enabled=true
// spring.flyway.locations=classpath:db/migration
// spring.flyway.baseline-on-migrate=true
```

**¿Por qué nunca modificar una migración existente?**
Flyway hashea cada archivo. Si cambias `V1__create_products_table.sql` después de haberse ejecutado, Flyway detecta el cambio y falla — el esquema en producción ya no coincide.

---

## 5. Jooq 3.19+ — SQL Type-Safe *(15 min)*

Jooq genera código Kotlin/Java desde el esquema de la base de datos. Si el esquema cambia, el código no compila hasta que lo regeneres.

```kotlin
// Jooq genera clases que representan las tablas
// Uso de la API type-safe
val productos = dslContext
    .select(
        PRODUCTOS.ID,
        PRODUCTOS.NOMBRE,
        PRODUCTOS.PRECIO
    )
    .from(PRODUCTOS)
    .where(PRODUCTOS.DISPONIBLE.isTrue)
    .and(PRODUCTOS.PRECIO.between(BigDecimal("100"), BigDecimal("500")))
    .orderBy(PRODUCTOS.NOMBRE.asc())
    .fetch()
    .map { record ->
        ProductoDto(
            id = record[PRODUCTOS.ID],
            nombre = record[PRODUCTOS.NOMBRE],
            precio = record[PRODUCTOS.PRECIO].toDouble()
        )
    }
```

**Ventaja sobre strings SQL:** Si renombras la columna `nombre` a `titulo` en la BD, el código `PRODUCTOS.NOMBRE` dejará de compilar — detectas el error antes de llegar a producción.

---

## 6. Testcontainers — Testing con BD Real *(10 min)*

En lugar de mockear la base de datos (que puede ocultar bugs), Testcontainers lanza un contenedor Docker real durante los tests:

```kotlin
@SpringBootTest
@Testcontainers
class ProductoRepositoryTest {

    companion object {
        @Container
        val postgres = PostgreSQLContainer<Nothing>("postgres:16-alpine").apply {
            withDatabaseName("test_db")
            withUsername("test")
            withPassword("test")
        }

        @DynamicPropertySource
        @JvmStatic
        fun configureProperties(registry: DynamicPropertyRegistry) {
            registry.add("spring.datasource.url", postgres::getJdbcUrl)
            registry.add("spring.datasource.username", postgres::getUsername)
            registry.add("spring.datasource.password", postgres::getPassword)
        }
    }

    @Autowired
    lateinit var productoRepository: ProductoRepository

    @Test
    fun `guardar y recuperar producto debe persistir correctamente`() {
        val producto = ProductoEntity(nombre = "Test", precio = BigDecimal("100"), categoria = "TEST")
        val guardado = productoRepository.save(producto)
        val recuperado = productoRepository.findById(guardado.id)
        assertTrue(recuperado.isPresent)
        assertEquals("Test", recuperado.get().nombre)
    }
}
```

---

## 7. Resumen de la Sesión

| Concepto | Clave a recordar |
|---|---|
| ORM vs SQL directo | ORM: más productivo, menos SQL manual. Jooq: SQL type-safe sin perder control. |
| Exposed DSL | Tablas como objects, queries en `transaction {}`. Type-safe total en Kotlin. |
| JPA + Hibernate | Estándar enterprise. `@Entity`, `@Repository`. Spring Data genera queries desde nombres. |
| Flyway | Migraciones como archivos SQL versionados `V1__nombre.sql`. Nunca modificar migración ejecutada. |
| Jooq | Genera código desde esquema BD. Error de compilación si el esquema cambia. |
| Testcontainers | BD real en Docker para tests. Detecta errores que los mocks ocultan. |

---

## Recursos Adicionales

| Recurso | Enlace | Para qué |
|---|---|---|
| Exposed Docs | [github.com/JetBrains/Exposed](https://github.com/JetBrains/Exposed) | ORM Kotlin de JetBrains |
| Spring Data JPA | [spring.io/guides/gs/accessing-data-jpa](https://spring.io/guides/gs/accessing-data-jpa/) | JPA con Spring Boot |
| Flyway Docs | [flywaydb.org/documentation](https://flywaydb.org/documentation/) | Migraciones de BD |
| Jooq Docs | [jooq.org/doc](https://www.jooq.org/doc/latest/manual/) | SQL type-safe |
| Testcontainers | [testcontainers.com](https://testcontainers.com/) | Testing con BD real en Docker |
| PostgreSQL Ranking | [db-engines.com/en/ranking](https://db-engines.com/en/ranking) | Popularidad de BD |
