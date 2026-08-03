# WEB_DESIGN_BRIEF.md — Prompt de diseño para Claude Design

> **Qué es esto:** el prompt para generar el diseño visual del dashboard de Jin (`Jin_Web`) en [claude.ai/design](https://claude.ai/design).
> **Cómo se usa:** copia todo lo que está debajo de la línea `═══` y pégalo como primer mensaje en Claude Design.
> **Fuente de verdad:** el inventario de funcionalidades sale del código real (`Jin_Core/src/tools/registry.ts`, `contracts/openapi.json`), no del blueprint — si el código cambia, este brief se regenera.
> **Estado:** escrito 2026-08-02, tras mergear Fase 6.1 (la API que el dashboard consume). Ownership de `Jin_Web`: Claude Code.

═══════════════════════════════════════════════════════════

# Diseña Jin — dashboard PWA de un sistema operativo personal con agentes de IA

## 1. Qué es el producto

**Jin** es un sistema operativo personal que orquesta agentes de IA autónomos. Los agentes leen correo, gestionan el calendario, siguen cursos universitarios, ejecutan código y levantan aplicaciones — pero **ninguna acción con consecuencias reales ocurre sin que un humano la apruebe**.

Este dashboard es la cabina de mando de ese sistema.

**Contexto crítico que cambia todas las decisiones de diseño:**

- **Un solo usuario.** Literalmente uno: el dueño del sistema. No hay registro, ni onboarding, ni roles, ni equipos, ni facturación, ni landing page. La primera pantalla es un login de una sola contraseña, y después estás dentro.
- **No es un SaaS.** No diseñes para convencer a nadie de nada. Diseña para alguien que ya confía en la herramienta y la usa 20 veces al día.
- **Es una interfaz de confianza, no de productividad.** El trabajo lo hacen los agentes. El trabajo del humano es *decidir si dejarlos*. Todo el diseño gira alrededor de esa decisión.

## 2. El problema de diseño central (lee esto dos veces)

El usuario recibe una tarjeta que dice: *"El agente quiere enviar este correo a tu profesor."*

Tiene que poder responder, **en dos segundos y desde el móvil, caminando**:

1. ¿Qué tan grave es esto si sale mal?
2. ¿Por qué el agente quiere hacerlo? ¿Qué leyó que lo llevó a proponerlo?
3. ¿Apruebo o rechazo?

**Un diseño que hace bonitos los botones de aprobar/rechazar pero no hace legible el riesgo, ha fracasado.** La jerarquía visual entera debe servir a esa pregunta.

Hay un peligro concreto y real: el sistema procesa contenido externo (correos, contenido de cursos) que podría contener instrucciones maliciosas intentando manipular al agente. Por eso cada tarjeta de aprobación **debe mostrar qué inputs externos influyeron en la decisión** — es una regla dura del sistema, no un extra. El usuario tiene que poder detectar "espera, esto lo propuso porque un correo raro se lo pidió".

## 3. Los 4 niveles de riesgo (el sistema de color/forma más importante)

Cada acción que un agente puede tomar tiene un nivel fijo:

| Nivel | Qué pasa | Ejemplos reales | Presencia en la UI |
|---|---|---|---|
| **auto** | Se ejecuta sola, sin avisar | leer correos, listar tareas del curso, listar eventos del calendario | Solo aparece en el historial |
| **notify** | Se ejecuta y avisa después | crear evento de calendario, agendar bloque de estudio, borrar evento *pasado* | Notificación post-hoc, sin acción requerida |
| **confirm** | Espera 1 aprobación | enviar correo, borrar evento *futuro*, ejecutar código, levantar una app, mergear código | **Tarjeta de aprobación** |
| **dual-confirm** | Requiere 2 aprobaciones separadas por 30 segundos obligatorios | acciones destructivas irreversibles | **Tarjeta + temporizador forzado** |

**Requisitos duros de este sistema visual:**

- Los 4 niveles deben distinguirse **sin depender solo del color** (forma, ícono, peso, borde). Alguien daltónico tiene que ver la diferencia entre "esto es rutina" y "esto es irreversible".
- La escalada de gravedad debe sentirse física: `auto` casi invisible → `dual-confirm` imposible de ignorar.
- **Los 30 segundos del dual-confirm son fricción intencional, no un cargando.** El diseño debe comunicar "esta pausa existe para que leas, no porque el sistema esté lento". Es la última barrera antes de algo irreversible. Nunca debe parecer un spinner ni un error.

## 4. Pantallas a diseñar

### 4.1 Login
Una contraseña. Nada más. Sin "recordarme", sin recuperación, sin registro. Debe verse deliberadamente austera — es la puerta de un sistema privado, no una app de consumo.

### 4.2 Overview (pantalla principal)
Estado del sistema de un vistazo:
- Aprobaciones pendientes (lo más prominente — es la razón de existir del dashboard)
- Consumo de presupuesto del día (tokens y dólares, con umbrales al 80% y 100%)
- Estado del "kill switch" (ver 4.5)
- Actividad reciente de los agentes

### 4.3 Bandeja de aprobaciones (HITL) — **la pantalla más importante**
Lista viva de acciones esperando decisión. Aparecen **en tiempo real** (llegan solas, sin recargar).

Cada tarjeta muestra:
- **Qué herramienta** quiere ejecutarse y su nivel de riesgo
- **Resumen del plan** en lenguaje natural ("Responder a Prof. Martínez confirmando asistencia")
- **El payload real, legible** — el correo completo, el evento exacto, el código que se va a correr. Sin esto el usuario aprueba a ciegas.
- **Inputs externos que influyeron** (obligatorio, ver sección 2)
- **Qué agente lo pidió** (puede haber varios agentes trabajando en paralelo)
- **Cuánto le queda antes de expirar** (las aprobaciones caducan a las 24h; el timeout nunca aprueba solo, solo descarta o escala)
- Aprobar / Rechazar

Diseña también: el estado vacío (nada pendiente — debería sentirse *bien*, no vacío), y el caso de 15 tarjetas acumuladas.

### 4.4 Chat con el agente
Conversación con el orquestador. El usuario pide objetivos en lenguaje natural ("revisa mi correo y arma mi agenda de mañana") y el agente trabaja.

Particularidad: **el agente muestra su plan antes de ejecutarlo** y va marcando pasos completados. Necesita una representación visual del plan en progreso dentro de la conversación — pasos pendientes, en curso, hechos, fallidos.

Nota técnica: la respuesta llega completa cuando el turno termina, **no token por token**. No diseñes un efecto de escritura en streaming.

### 4.5 Presupuesto y kill switch
- Consumo por sesión / día (tokens y dólares) contra sus límites
- Alertas al 80% y 100%
- **Kill switch**: si el sistema detecta consumo desbocado (más de 2× lo normal en una hora), congela *todos* los agentes automáticamente. Esta pantalla muestra ese estado y permite reanudar — y reanudar es en sí una acción que requiere aprobación.
- El estado "kill switch activo" debe ser visible desde **cualquier** pantalla, no solo esta. Es una condición de emergencia.

### 4.6 Audit log
Registro inmutable y encadenado criptográficamente de cada acción del sistema. Tabla paginada, con filtros. Muestra: momento, quién actuó (sistema / usuario / agente específico), qué herramienta, estado de aprobación, y los hashes de la cadena.

Es forense: denso, monoespaciado donde corresponde, hecho para escanear y buscar, no para lucir bonito.

### 4.7 Memoria
Buscador sobre la memoria a largo plazo del agente (hechos sobre el usuario, preferencias, lecciones de sesiones pasadas). Búsqueda semántica: escribes una consulta, salen recuerdos relevantes con su origen y fecha.

### 4.8 Board de orquestación multi-agente
Cuando un objetivo es complejo, el orquestador lo descompone en tickets y los reparte entre varios sub-agentes que trabajan en paralelo. **Es literalmente un tablero estilo Jira**: tickets con estado (pendiente / en curso / hecho / fallido / bloqueado), dependencias entre ellos, y un hilo de comentarios por ticket donde los agentes se responden entre sí.

Detalle importante: cuando dos agentes se contradicen, **el conflicto se registra visiblemente, no se esconde**. El tablero debe mostrar esos conflictos y si los resolvió el orquestador solo o si escalaron al humano.

### 4.9 Editor de código con comentarios de IA
Editor tipo VS Code (Monaco) donde la IA comenta línea por línea mediante widgets insertados entre líneas de código. Cada comentario se acepta o descarta individualmente.

### 4.10 Preview apps
Los agentes pueden levantar aplicaciones reales corriendo (un frontend, un backend de prueba) accesibles por URL pública temporal. Esta pantalla lista esas apps corriendo, con su tiempo de vida restante (expiran obligatoriamente, máximo 24h) y permite detenerlas.

Se ven embebidas en iframes con aislamiento estricto — visualmente debe quedar claro que **ese contenido lo generó una IA y no es parte confiable del sistema**. Un marco, una etiqueta, algo que lo separe.

## 5. Requisitos de PWA

Se instala en el móvil desde el navegador y vive en la pantalla de inicio como una app nativa.

- **Modo standalone**: sin barra del navegador. Necesita su propia navegación y respetar las áreas seguras (notch, barra inferior).
- **Ícono e identidad de marca** para la pantalla de inicio y la pantalla de arranque.
- **Estado offline**: solo lectura degradada. Muestra claramente "sin conexión" y qué datos son de la última sincronización. **Nunca** permitas aprobar o rechazar sin conexión — una aprobación en cola que se sincroniza 3 horas tarde es un problema de seguridad, no una comodidad. Diseña ese bloqueo explícitamente.
- **Notificaciones push** (a futuro): diseña la pantalla de permisos y ajustes, aunque el backend aún no las envía. Sería el reemplazo natural del bot de Telegram que hoy usa el dueño para aprobar desde el móvil.

**Prioridad por dispositivo:**
- **Móvil primero**: aprobaciones, chat, presupuesto, overview. Es donde se aprueba con el celular en la mano.
- **Escritorio primero**: editor de código, audit log, board de orquestación. Densos, necesitan pantalla.

## 6. Dirección visual

- **Oscuro por defecto**, con modo claro funcional. Es una herramienta de operación que se usa de noche.
- **Densa pero respirable.** Es una cabina, no una landing. Sin héroes vacíos, sin degradados de marketing, sin ilustraciones decorativas.
- **Monoespaciado** para código, hashes, identificadores y payloads. Proporcional para el resto.
- **Tiempo real visible**: cosas que aparecen solas necesitan una transición que las haga notar sin sobresaltar.
- Personalidad: precisa, sobria, con carácter. Piensa en herramientas de infraestructura serias (Linear, Vercel, Grafana), no en dashboards de startup genéricos.
- **El nombre es Jin.** Corto, se presta a un ícono fuerte. No hay identidad de marca previa — proponla.

## 7. Qué NO diseñar

- Onboarding, tour, tutoriales, estados de bienvenida
- Landing, pricing, marketing
- Gestión de usuarios, roles, permisos, invitaciones, perfiles
- Configuración extensa — casi todo se configura por archivos en el servidor
- Modo multi-cuenta o multi-tenant

## 8. Entregable

Sistema de diseño + pantallas clave:
1. **Fundamentos**: color (incluido el sistema de los 4 niveles de riesgo), tipografía, espaciado, íconos
2. **Componentes**: tarjeta de aprobación en sus 4 niveles y sus estados (pendiente, temporizador de dual-confirm, aprobada, rechazada, expirada), burbujas de chat, plan en progreso, fila de audit log, medidor de presupuesto, ticket del board, banner de kill switch, estado offline
3. **Pantallas**: login, overview, bandeja de aprobaciones (móvil y escritorio), chat, presupuesto, audit log, board, editor
4. Estados vacíos, de carga y de error para cada pantalla

Empieza por la **tarjeta de aprobación en sus 4 niveles** — si ese componente funciona, el resto del sistema se ordena solo detrás de él.
