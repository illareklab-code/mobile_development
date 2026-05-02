# CodLab N°5 — Figma y Definición del Proyecto del Curso
**Curso:** Desarrollo de Sistemas Móviles | **Semana:** 5  
**Evaluación:** Individual + Grupal

---

## Objetivos

- Crear un diseño completo de 3 pantallas en Figma.
- Aplicar el sistema de colores Material Design 3 con Material Theme Builder.
- Definir el equipo y el concepto del proyecto del curso.
- Practicar Auto Layout, Componentes y prototipado básico.

## Prerrequisitos

- Cuenta en Figma creada (figma.com).
- Leer la teoría de Semana 5.
- Equipo de 3–4 estudiantes formado.

---

## Parte 1 — Figma (Individual, 70 min)

### Paso 1: Crear el archivo de diseño

1. Inicia sesión en Figma → **New File** → renombrar a `DSM_CodLab5_TuApellido`.
2. Configura una página llamada "Pantallas".

### Paso 2: Configurar la paleta de colores con Material Theme Builder

1. Abre [m3.material.io/theme-builder](https://m3.material.io/theme-builder) en una pestaña nueva.
2. Selecciona un color primario que represente el concepto de tu proyecto (o usa #6750A4 si no tienes uno).
3. Genera la paleta → **Export** → **Figma** → copia el link del kit de Figma generado.
4. En tu archivo Figma: duplica el kit de M3 a tu proyecto o importa los estilos de color.

### Paso 3: Crear la pantalla de Bienvenida/Login

Crea un Frame (F) → selecciona preset "Android Small" (360×800).

Diseña una pantalla de bienvenida/login con:
- **Logo** de la app (puedes usar un ícono de Figma Community o texto grande).
- **Título** con el nombre de la app.
- **Subtítulo** de descripción breve.
- **Botón** "Iniciar sesión" (Filled Button — 48dp altura mínima).
- **Botón** "Registrarse" (Outlined Button).
- **Link** "¿Olvidaste tu contraseña?".

Usa Auto Layout (Shift+A) en todo el contenido principal:
- Dirección: Vertical.
- Padding: 24dp (arriba, abajo, izquierda, derecha).
- Gap entre elementos: 16dp.

### Paso 4: Crear la pantalla de Lista Principal

Nuevo Frame (360×800). Diseña:
- **TopAppBar** con título de la sección.
- **SearchBar** (TextField con ícono de búsqueda a la izquierda).
- **Fila de chips** de filtro (al menos 3 categorías).
- **Lista de cards** (mínimo 4 cards):
  - Cada card: imagen/ícono + título + descripción + precio o badge.
  - Usa Auto Layout en cada card.
- **FAB** (Floating Action Button) en la esquina inferior derecha.

Crea el card como un **Componente** (Ctrl+Alt+K):
1. Diseña el card una sola vez.
2. Conviértelo en componente.
3. Instancia el componente 4 veces para la lista.
4. Cambia el contenido de cada instancia (overrides).

### Paso 5: Crear la pantalla de Detalle

Nuevo Frame (360×800). Diseña:
- **Imagen/banner** grande en la parte superior (160dp altura).
- **TopAppBar** con botón de regreso (← ícono).
- **Título** del elemento.
- **Descripción** completa (texto largo de prueba — Lorem ipsum en español).
- **Chips** de etiquetas/categorías.
- **Botón principal** de acción (Ej: "Agregar al carrito", "Ver más", "Contactar").

### Paso 6: Conectar pantallas con prototipado

1. Ve a la pestaña **Prototype** (barra superior derecha).
2. Conecta el botón "Iniciar sesión" → pantalla de Lista.
3. Conecta uno de los cards → pantalla de Detalle.
4. Conecta el botón de regreso (←) → Lista.
5. Haz clic en ▶ **Present** (arriba a la derecha) para ver el flujo interactivo.

### Paso 7: Inspeccionar con Dev Mode

1. Activa Dev Mode (botón `</>` arriba a la derecha).
2. Selecciona el botón "Iniciar sesión" → anota:
   - Padding (en dp)
   - Color del fondo (hex)
   - Tipografía (tamaño y peso)
3. Comparte el link del archivo Figma (Edit mode) con el docente.

---

## Parte 2 — Definición del Proyecto (Grupal, 30 min)

### Paso 8: Definir el proyecto del curso

Con tu equipo, completa el **Documento de Inicio del Proyecto** en un archivo de texto o Notion:

```
## Proyecto DSM 2026 — Nombre del equipo

### App: [Nombre de la app]
### Descripción: [1 párrafo — qué hace la app]
### Usuario objetivo: [quién la usa y para qué]

### 3 funcionalidades principales:
1. [Feature 1 — ej: "Gestión de inventario con búsqueda"]
2. [Feature 2 — ej: "Escáner de código QR para productos"]
3. [Feature 3 — ej: "Dashboard con estadísticas de ventas"]

### Stack tecnológico planificado:
- Frontend: Kotlin + Jetpack Compose
- Backend: [Spring Boot / Ktor / FastAPI]
- Base de datos cloud: Firebase Firestore
- Base de datos local: Room
- Autenticación: Firebase Auth

### Integrantes del equipo:
| Nombre | Rol | GitHub |
|---|---|---|
| [Nombre] | Team Lead / Dev | @usuario |
| [Nombre] | Backend Dev | @usuario |
| [Nombre] | Frontend Dev | @usuario |
| [Nombre] | Full Stack | @usuario |

### Repositorio GitHub: [link al repo creado]
```

### Paso 9: Crear el repositorio del proyecto

1. Un integrante crea un repositorio en GitHub: `dsm-2026-[nombre-app]`.
2. Configura el repo:
   - `README.md` con la descripción del proyecto (pueden copiar el documento de inicio).
   - `.gitignore` para Android (GitHub lo genera automáticamente).
   - Crea una branch `develop` desde `main`.
3. Invita a todos los integrantes como colaboradores.
4. Crea un proyecto en GitHub Projects con columnas: `Backlog`, `En progreso`, `Listo`.

---

## Entregables

**Individual:**
- Link al archivo Figma (compartir como "Can view" al docente: john.barrera@unmsm.edu.pe).
- 3 pantallas diseñadas con Auto Layout y colores M3.
- Prototipo funcional conectando las 3 pantallas.

**Grupal:**
- Documento de inicio del proyecto completado.
- Repositorio GitHub creado con todos los integrantes.
- Primeras 5 historias de usuario creadas en GitHub Projects.

## Criterios de Evaluación

| Criterio | Puntaje |
|---|---|
| 3 pantallas en Figma con Auto Layout | 3 pts |
| Colores de M3 aplicados consistentemente | 1 pt |
| Card implementado como componente reutilizable | 1 pt |
| Prototipo conecta las 3 pantallas | 2 pts |
| Documento de proyecto completado (grupal) | 2 pts |
| Repositorio GitHub con colaboradores | 1 pt |
| **Total** | **10 pts** |
