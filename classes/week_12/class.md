# Semana 12 — Exposición Sprint 2
**Curso:** Desarrollo de Sistemas Móviles | **Unidad:** U3  
**Tipo:** Evaluación grupal | **Ponderación:** Parte de N2 (Evaluación Continua — 40%)

---

## Evaluación

Los equipos presentan el **Sprint 2** del proyecto del curso.

---

## Requisitos de la Presentación

- **Duración:** 10 minutos por equipo + 5 minutos de preguntas.
- **Demo en vivo** con el backend corriendo (local con ngrok, o desplegado en Railway/Render).
- Mostrar el flujo completo: login → listar → crear → eliminar.
- Recorrido rápido de la arquitectura del backend (diagrama o explicación verbal).

---

## Alcance del Sprint 2 (Semanas 9–11)

| Requisito | Descripción |
|---|---|
| Autenticación | Firebase Auth (email/password o Google) con persistencia de sesión |
| Base de datos local | Room con al menos 1 entidad y consultas reactivas con Flow |
| Preferencias | DataStore para al menos 2 configuraciones de usuario |
| Backend REST | Spring Boot / Ktor con mínimo 5 endpoints CRUD documentados en Swagger |
| ORM backend | Spring Data JPA o Exposed con base de datos PostgreSQL o H2 |
| Integración Android | App consume el backend con Retrofit + JWT en headers |
| Pantallas | Mínimo 4 pantallas con navegación type-safe |

---

## Criterios de Evaluación

| Criterio | Peso |
|---|---|
| Backend API funcionando (5+ endpoints) | 25% |
| Integración Android ↔ Backend | 25% |
| Persistencia local (Room + DataStore) | 20% |
| Autenticación Firebase con pantalla de login | 20% |
| Calidad del código + Git | 10% |

### Evaluación individual
Se preguntará a cada integrante sobre una parte específica del código. La nota puede diferir entre miembros del equipo.

---

## Contexto Técnico para Preguntas

El docente puede preguntar sobre los temas de Unit 3 vistos hasta esta semana:

**Testing avanzado 2026 (contexto para Q&A):**
- Integration testing con Testcontainers: BD real en Docker durante los tests.
- Contract testing con Spring Cloud Contract o Pact: verificar que el cliente Android y el servidor hablan el mismo "lenguaje".
- CI/CD con GitHub Actions: los tests corren automáticamente en cada push.
- Quality gates con SonarQube: cobertura de código, code smells, vulnerabilidades.

**CI/CD que deberían tener para el Sprint 3:**
- Pipeline en GitHub Actions: lint + build + test en cada push.
- Deploy automático al aprobar un PR en `main`.
- SBOM (Software Bill of Materials): lista de todas las dependencias del proyecto.

---

## Checklist de Preparación

- [ ] Backend desplegado o corriendo con ngrok/localhost tunneling.
- [ ] App Android configurada con la URL del backend desplegado.
- [ ] Todos los integrantes pueden explicar la capa de datos de su módulo.
- [ ] Repositorio GitHub actualizado con commits de cada semana.
- [ ] Firebase Security Rules protegen los datos por usuario autenticado.
- [ ] Swagger UI del backend accesible para mostrar al docente.
- [ ] No hay API keys ni passwords en el código — usar variables de entorno.
