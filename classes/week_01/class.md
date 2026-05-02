# Semana 1 — Fundamentos de Aplicaciones Móviles
**Curso:** Desarrollo de Sistemas Móviles | **Unidad:** U1  
**Sesión:** Presencial | **Duración:** 2 horas

---

## Objetivos de la sesión

Al finalizar la clase, el estudiante será capaz de:
- Explicar qué es una aplicación móvil y sus tipos principales.
- Describir la evolución del ecosistema móvil desde sus inicios hasta 2026.
- Distinguir las arquitecturas de Android e iOS.
- Identificar el stack tecnológico relevante para el curso.

---

## 1. Presentación del Sílabo *(15 min)*

### Dinámica de apertura
Pregunta al grupo: *¿Cuántas apps abriste hoy antes de llegar a clase?*  
El promedio global es **10 apps por usuario al día** ([App Annie / data.ai, State of Mobile 2024](https://www.data.ai/en/go/state-of-mobile-2024/)).

### Puntos clave del sílabo
- **Unidades:** 4 unidades progresivas (Fundamentos → UI/HTTP → BD/Backend → Seguridad/Despliegue).
- **Evaluación:** N1 30% (Sprint 1, Semana 8) · N2 40% (Continua) · N3 30% (Expo final, Semana 16).
- **Proyecto del curso:** App móvil completa en equipos, presentada en 3 sprints.
- **CodLabs:** Evaluación individual, presentación en clase.
- **Asistencia mínima:** 70% para ser evaluado.

### Stack del curso 2026
El curso se desarrolla en **Kotlin + Jetpack Compose** como lenguaje y framework principal, con **Spring Boot 3.3+** en el backend y **Firebase/Firestore** como servicio en la nube.

---

## 2. Fundamentos de Aplicaciones Móviles *(25 min)*

### 2.1 ¿Qué es una aplicación móvil?

Una aplicación móvil es software diseñado para ejecutarse en dispositivos portátiles (smartphones, tablets, wearables, etc.) bajo restricciones de:
- **Recursos limitados:** batería, RAM, CPU térmicamente restringida.
- **Conectividad variable:** redes 5G, Wi-Fi, o sin conexión.
- **Interacción táctil:** gestos, voz, sensores.
- **Ciclo de vida no lineal:** el SO puede pausar o matar procesos en cualquier momento.

### 2.2 Tipos de aplicaciones móviles

| Tipo | Descripción | Ejemplos | Ventajas | Desventajas |
|---|---|---|---|---|
| **Nativa** | Escrita para un SO específico (Kotlin/Android, Swift/iOS) | Google Maps, WhatsApp | Máximo rendimiento, acceso total al hardware | Código separado por plataforma |
| **Multiplataforma** | Un código base para Android e iOS | Flutter, KMM | Reutilización de código | Overhead de abstracción |
| **Web progresiva (PWA)** | Web app instalable desde el navegador | Twitter Lite, Starbucks | Sin tienda de apps | Acceso limitado al hardware |
| **Híbrida** | WebView dentro de un contenedor nativo | Ionic | Desarrollo rápido | Rendimiento inferior a nativa |

> **Tendencia 2026:** el 62% de los equipos de desarrollo nuevos elige Flutter o Kotlin Multiplatform sobre desarrollo nativo separado, según el [Developer Survey de Stack Overflow 2024](https://survey.stackoverflow.co/2024/technology#most-popular-technologies-misc-tech).

### 2.3 El mercado móvil en cifras

- **Descargas anuales (2023):** 257,000 millones de apps descargadas a nivel global ([data.ai State of Mobile 2024](https://www.data.ai/en/go/state-of-mobile-2024/)).
- **Ingresos de apps (2023):** USD 171,000 millones entre App Store y Google Play ([Sensor Tower, Store Intelligence](https://sensortower.com/blog/app-revenue-and-download-estimates)).
- **Tiempo en pantalla:** el usuario promedio pasa **4 h 37 min al día** en su smartphone ([data.ai State of Mobile 2024](https://www.data.ai/en/go/state-of-mobile-2024/)).

---

## 3. Evolución de los Sistemas Móviles *(20 min)*

### Línea de tiempo

```
1994  ── IBM Simon: primer smartphone (pantalla táctil, email, apps)
2007  ── iPhone OS 1.0: interfaz multi-touch, sin App Store todavía
2008  ── Android 1.0 (HTC Dream) + App Store iOS lanzado
2009  ── Google Play Store (Android Market) abierto
2011  ── iOS 5 con iCloud; Android 4.0 ICS unifica teléfono/tablet
2014  ── Android 5.0 Lollipop: Material Design 1.0
2017  ── Kotlin declarado lenguaje oficial para Android
2019  ── Flutter 1.0 estable; SwiftUI para iOS
2021  ── Jetpack Compose 1.0 estable (UI declarativa para Android)
2023  ── LLMs on-device (Gemini Nano, Phi-2); visionOS para Apple Vision Pro
2025  ── Kotlin 2.0 + KMM enterprise; Flutter 4.x; Android 16; IA on-device generalizada
2026  ── [Hoy] Compose Adaptive, foldables mainstream, LLMs integrados nativamente
```

### Hitos clave que cambiaron el paradigma

| Año | Evento | Impacto |
|---|---|---|
| 2007 | iPhone OS | La pantalla táctil capacitiva reemplaza el teclado físico |
| 2008 | App Store / Android Market | El modelo de distribución digital masivo nace |
| 2014 | Material Design | Diseño sistémico para apps Android (coherencia visual) |
| 2017 | Kotlin oficial | Java queda en segundo plano para Android |
| 2021 | Jetpack Compose 1.0 | UI declarativa reemplaza XML layouts en Android |
| 2023 | IA on-device generalizada | Los modelos ligeros corren directo en el teléfono |

---

## 4. Sistemas Operativos: Android e iOS *(30 min)*

### 4.1 Cuotas de mercado

A mayo 2026, la distribución global de SO móviles es:
- **Android:** ~71.8%
- **iOS:** ~27.6%
- **Otros:** < 1%

Fuente: [StatCounter Global Stats — Mobile OS Market Share Worldwide](https://gs.statcounter.com/os-market-share/mobile/worldwide/)

> En Latinoamérica, Android supera el **85%** de cuota de mercado. Para el mercado peruano, desarrollo Android es la habilidad de mayor demanda.

---

### 4.2 Arquitectura de Android

```
┌─────────────────────────────────────────────┐
│            APLICACIONES (tu capa)           │  ← Kotlin + Jetpack Compose
├─────────────────────────────────────────────┤
│         FRAMEWORK DE APLICACIÓN             │  ← Activity Manager, Package Manager,
│                                             │     Window Manager, Content Providers
├─────────────────────────────────────────────┤
│        BIBLIOTECAS NATIVAS + ART            │  ← Android Runtime (ART), OpenGL, SQLite,
│                                             │     WebKit, libc
├─────────────────────────────────────────────┤
│           CAPA DE ABSTRACCIÓN               │  ← HAL (Hardware Abstraction Layer)
├─────────────────────────────────────────────┤
│               KERNEL LINUX                  │  ← Drivers: Cámara, WiFi, Bluetooth,
│                                             │     Binder IPC, Power Management
└─────────────────────────────────────────────┘
```

**Capas clave que debes conocer:**

| Capa | Relevancia para el desarrollador |
|---|---|
| **ART (Android Runtime)** | Compila bytecode Kotlin/Java a código máquina (AOT + JIT). Reemplazó a Dalvik en Android 5+. |
| **Jetpack Libraries** | Conjunto de bibliotecas curadas por Google (Compose, Room, Navigation, etc.) que siguen buenas prácticas. |
| **HAL** | Permite que el mismo SO funcione en hardware de distintos fabricantes (Samsung, Xiaomi, Google Pixel). |
| **Kernel Linux** | Gestión de procesos, memoria, seguridad. Android adapta el kernel con extensiones propias (Binder IPC). |

**Versiones relevantes en 2026:**

| Versión | Nombre | API Level | Cuota aprox. |
|---|---|---|---|
| Android 16 | - | 36 | ~15% (lanzamiento 2025) |
| Android 15 | - | 35 | ~30% |
| Android 14 | - | 34 | ~35% |
| Android 13 | Tiramisu | 33 | ~15% |
| Android ≤12 | - | ≤32 | ~5% (en declive) |

Fuente: [Android Distribution Dashboard](https://developer.android.com/about/dashboards)

> **Para el curso:** el `minSdk` recomendado es **API 24 (Android 7.0)** para cubrir >99% de dispositivos activos. El `targetSdk` debe ser siempre el más reciente (API 35+).

---

### 4.3 Arquitectura de iOS

```
┌─────────────────────────────────────────────┐
│             COCOA TOUCH / UIKIT             │  ← Swift + SwiftUI / UIKit
├─────────────────────────────────────────────┤
│                   MEDIA                     │  ← Core Audio, Core Video, AVFoundation
├─────────────────────────────────────────────┤
│               CORE SERVICES                 │  ← Core Data, Core Location, CloudKit,
│                                             │     Core Motion, HealthKit
├─────────────────────────────────────────────┤
│                 CORE OS                     │  ← Darwin kernel (XNU), Security Framework,
│                                             │     Accelerate, Local Auth, Bluetooth LE
└─────────────────────────────────────────────┘
```

**Diferencias clave vs Android:**

| Aspecto | Android | iOS |
|---|---|---|
| Kernel | Linux (adaptado) | Darwin/XNU (Unix BSD) |
| Lenguaje principal | Kotlin | Swift |
| UI Framework | Jetpack Compose | SwiftUI |
| Distribución de apps | Google Play + sideloading | App Store exclusivo (salvo EU) |
| Fragmentación | Alta (miles de dispositivos) | Baja (hardware propio Apple) |
| Proceso de publicación | Revisión 1-3 días | Revisión 1-7 días, más estricto |

---

## 5. Ecosistema Móvil 2026 *(15 min)*

### 5.1 Nuevas plataformas que debes conocer

| Plataforma | SO | Relevancia |
|---|---|---|
| **Foldables** (Galaxy Z, Pixel Fold) | Android 15+ | Requieren diseño adaptativo para 2-3 estados de pantalla |
| **Wear OS 5.x** | Android Wear | Wearables con Compose for Wear OS |
| **Apple Vision Pro** | visionOS | Desarrollo 3D/espacial con SwiftUI |
| **Android Auto / CarPlay** | Android / iOS | Interfaces simplificadas para conductores |

### 5.2 IA integrada en el desarrollo móvil

En 2026, la IA no es opcional — está integrada en toda la cadena:

**En el IDE (durante el desarrollo):**
- **GitHub Copilot Enterprise** para Android Studio → autocompleción con contexto del proyecto.
- **Gemini Code Assist** integrado en Android Studio Meerkat.

**En la app (en tiempo de ejecución):**
- **ML Kit** de Google → visión computacional, detección de texto, traducción on-device.
- **Gemini Nano** → LLM on-device en Pixel 8+ y Galaxy S24+. Corre sin internet.
- **Claude API** (Anthropic) → para features que requieren razonamiento más complejo desde la nube.
- **Jetpack Generative AI Kit** → wrapper oficial de Google para integrar modelos IA en apps Android.

> **Estadística:** el 78% de las apps top-100 de Google Play incluye al menos una feature de IA o ML en 2025 ([Google I/O 2025, keynote Android](https://io.google/2025/explore/)).

### 5.3 ¿Por qué Kotlin + Compose es el estándar en 2026?

1. **Kotlin** es el lenguaje oficial de Android desde 2017. En 2025, el **95% del código nuevo en Android es Kotlin** ([Google Android Developers Blog](https://android-developers.googleblog.com/)).
2. **Jetpack Compose** reemplazó a XML layouts como sistema de UI estándar desde Compose 1.0 (2021). En 2026, Google ya no actualiza el sistema de vistas XML.
3. **Interoperabilidad:** Kotlin Multiplatform (KMM) permite compartir lógica de negocio entre Android e iOS sin perder el rendimiento nativo.

---

## 6. Resumen de la sesión

| Concepto | Clave a recordar |
|---|---|
| App móvil | Software con restricciones de batería, conectividad y ciclo de vida gestionado por el SO |
| Tipos de apps | Nativa > rendimiento; Multiplataforma > productividad; PWA > web |
| Android | ~72% del mercado mundial; arquitectura en capas sobre kernel Linux; fragmentación alta |
| iOS | ~28% mundial; ecosistema cerrado y controlado; baja fragmentación |
| 2026 | Jetpack Compose + Kotlin estándar; IA on-device generalizada; foldables y wearables en auge |

---

## 7. Tarea previa al CodLab

Antes de la sesión de laboratorio:

1. Instalar **Android Studio Meerkat** (última versión estable) desde [developer.android.com/studio](https://developer.android.com/studio).
2. Verificar que el emulador funciona ejecutando el `Hello World` por defecto.
3. Registrar una cuenta en [Firebase Console](https://console.firebase.google.com/) con tu correo institucional.
4. Crear cuenta en [GitHub](https://github.com) — todo el código del curso se entrega por repositorio.

---

## 8. Recursos complementarios

| Recurso | Enlace | Para qué |
|---|---|---|
| Android Developers — Guía oficial | [developer.android.com/guide](https://developer.android.com/guide) | Referencia base del curso |
| Kotlin Docs | [kotlinlang.org/docs](https://kotlinlang.org/docs/home.html) | Sintaxis y características del lenguaje |
| Jetpack Compose Docs | [developer.android.com/compose](https://developer.android.com/jetpack/compose/documentation) | Framework UI del curso |
| StatCounter OS Market Share | [gs.statcounter.com/os-market-share/mobile](https://gs.statcounter.com/os-market-share/mobile/worldwide/) | Cuotas de mercado actualizadas |
| Stack Overflow Survey 2024 | [survey.stackoverflow.co/2024](https://survey.stackoverflow.co/2024/) | Popularidad de tecnologías |
| data.ai State of Mobile 2024 | [data.ai/state-of-mobile-2024](https://www.data.ai/en/go/state-of-mobile-2024/) | Estadísticas de la industria móvil |
| Flutter vs React Native vs KMM | [jetbrains.com/lp/devecosystem-2024](https://www.jetbrains.com/lp/devecosystem-2024/) | Comparativa de frameworks 2024 |
