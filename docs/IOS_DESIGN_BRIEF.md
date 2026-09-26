# IOS_DESIGN_BRIEF.md — Prompt de diseño para Claude Design (app iOS)

> **Qué es esto:** el prompt para diseñar la app nativa de Jin para iPhone en [claude.ai/design](https://claude.ai/design). Es el sucesor móvil de `WEB_DESIGN_BRIEF.md`, pero **no es un port del dashboard web**: cada funcionalidad está repensada para iOS.
> **Cómo se usa:** copia todo lo que está entre las dos líneas `═══` y pégalo como primer mensaje en Claude Design. La sección final ("Notas para implementación") es para el owner y Claude Code — no la pegues.
> **Fuente de verdad:** el inventario sale del código real a 2026-09-25 (`Jin_Core/contracts/openapi.json`, `src/tools/registry.ts`, `src/chat/chat.gateway.ts`, `src/realtime/realtime.gateway.ts`) y de las pantallas vivas de `Jin_Web`. Incluye el streaming del chat de Jin_Core#50/#51 (en revisión). Si el código cambia, este brief se regenera.
> **Estado:** borrador 2026-09-25. El owner no quiere conservar el diseño visual actual del dashboard web — la dirección visual queda abierta.

═══════════════════════════════════════════════════════════

# Diseña Jin para iPhone — app nativa iOS de un sistema operativo personal con agentes de IA

## 1. Qué es el producto

**Jin** es un sistema operativo personal que orquesta agentes de IA autónomos. Los agentes leen correo, gestionan el calendario, siguen cursos universitarios (Canvas), ejecutan código, levantan aplicaciones web temporales y coordinan varios sub-agentes entre sí — pero **ninguna acción con consecuencias reales ocurre sin que un humano la apruebe**.

Esta app es la cabina de mando de ese sistema, en el bolsillo.

**Contexto que cambia todas las decisiones de diseño:**

- **Un solo usuario.** Literalmente uno: el dueño del sistema. Sin registro, sin onboarding, sin roles, sin equipos, sin facturación. Se entra con una contraseña y después está adentro.
- **No es una app de consumo.** No hay que convencer a nadie. Es para alguien que ya confía en la herramienta y la abre 20 veces al día, muchas veces caminando, con una mano.
- **Es una interfaz de confianza, no de productividad.** El trabajo lo hacen los agentes. El trabajo del humano es *decidir si dejarlos* y *ver qué están haciendo*.
- **Hoy el dueño usa un bot de Telegram para aprobar desde el móvil.** La app tiene que ser claramente mejor que eso — si no, no tiene razón de existir.

## 2. Por qué una app nativa (qué falló en la versión web móvil)

El dueño usó el dashboard web instalado como PWA en su iPhone 16 Pro Max. Estos problemas reales son requisitos de este diseño:

1. **La barra de navegación inferior no quedaba fija** — había que scrollear para llegar a ella. En iOS se usa la tab bar nativa; nunca navegación que dependa del scroll.
2. **Al cambiar de pestaña se perdían los mensajes del chat**, y Jin no tenía contexto de lo que se le había dicho antes. La conversación tiene que sobrevivir a cambiar de pestaña, cerrar la app y volver.
3. **Jin "no respondía"**: mandaba un pedido y durante mucho rato no veía nada. Ahora el agente transmite su trabajo en vivo (ver 6.4) — el diseño tiene que hacer visible que Jin está trabajando y en qué.
4. **No sabía qué modelo de IA le respondía** (Haiku, Sonnet u Opus) ni cuántos tokens gastaba. Tiene que estar a la vista sin ser ruido.
5. **Cuando el proveedor de IA rechaza un pedido**, Jin mostraba un mensaje vacío. Ahora devuelve un aviso explícito; el diseño necesita un estado para "el modelo no quiso/pudo responder" distinto de un error de red.

## 3. El problema de diseño central (lee esto dos veces)

El usuario recibe: *"El agente quiere enviar este correo a tu profesor."*

Tiene que poder responder **en dos segundos, con el celular en una mano, caminando**:

1. ¿Qué tan grave es si sale mal?
2. ¿Por qué el agente quiere hacerlo? ¿Qué leyó que lo llevó a proponerlo?
3. ¿Apruebo o rechazo?

**Un diseño que hace bonitos los botones de aprobar/rechazar pero no hace legible el riesgo, fracasó.**

Peligro concreto: el sistema procesa contenido externo (correos, contenido de cursos) que puede traer instrucciones maliciosas para manipular al agente (prompt injection). Por eso cada aprobación **muestra qué inputs externos influyeron en la decisión** ("INFLUIDO POR: readEmails (1)"). Es una regla dura, no un extra: el usuario tiene que poder detectar "espera, esto lo propuso porque un correo raro se lo pidió".

## 4. Los 4 niveles de riesgo (el sistema visual más importante)

Cada acción que un agente puede tomar tiene un nivel fijo:

| Nivel | Qué pasa | Acciones reales hoy | Presencia en la app |
|---|---|---|---|
| **auto** | Se ejecuta sola, sin avisar | leer correos, listar eventos del calendario, listar tareas y contenido de cursos, buscar en el corpus de documentos, consultar documentación técnica, listar apps corriendo | Solo en el historial / en el plan en vivo del chat |
| **notify** | Se ejecuta y avisa después | crear o editar evento de calendario, agendar bloque de estudio, borrar evento *pasado*, detener una app | Notificación post-hoc, sin acción requerida |
| **confirm** | Espera 1 aprobación | enviar correo, borrar evento *futuro*, ejecutar código, levantar una app pública, mergear código de un agente, resolver un conflicto entre agentes | **Tarjeta de aprobación** |
| **dual-confirm** | 2 aprobaciones separadas por **30 s obligatorios** | bajar la protección del sistema (activar los modos de autonomía, ver 6.6) y acciones irreversibles | **Tarjeta + espera forzada** |

**Requisitos duros:**

- Los 4 niveles se distinguen **sin depender solo del color** (forma, ícono, peso, borde, texto). Alguien daltónico tiene que distinguir "rutina" de "irreversible".
- La gravedad escala físicamente: `auto` casi invisible → `dual-confirm` imposible de ignorar.
- **Los 30 s del dual-confirm son fricción intencional, no un "cargando".** Tiene que comunicar "esta pausa existe para que releas", nunca parecer un spinner ni un error. Es la última barrera antes de algo irreversible.

## 5. Arquitectura de la app

### 5.1 Navegación

Tab bar nativa, siempre visible, alcanzable con el pulgar:

1. **Inicio** — estado del sistema de un vistazo.
2. **Aprobar** — bandeja de aprobaciones, con badge del número pendiente.
3. **Chat** — conversación con el orquestador.
4. **Gasto** — presupuesto y kill switch.
5. **Más** — Autonomía, Claude Code, Board de orquestación, Apps, Memoria, Audit, Ajustes.

Propón si alguna de las secundarias merece subir a la tab bar (ej. Claude Code, si el dueño la usa mucho) o si "Más" debe ser una lista, una grilla o un sheet.

### 5.2 Estados globales (visibles desde CUALQUIER pantalla)

Son condiciones que no pueden quedar escondidas dentro de una pestaña:

- **Kill switch activo** — los agentes están congelados por consumo anómalo. Emergencia.
- **Modo de autonomía relajado** (semiautomático/automático) — el agente está ejecutando acciones sin pedir permiso, con cuenta regresiva hasta volver solo a supervisado.
- **Estado de conexión** (4 estados, ver sección 7).

Pueden convivir los tres a la vez. Diseña cómo se apilan sin tapar el contenido ni comerse la pantalla (y considera el Dynamic Island / zona superior).

### 5.3 Tiempo real

La app recibe eventos en vivo por WebSocket: aprobación nueva, mensaje nuevo de Claude Code, alerta de presupuesto (80%/100%), kill switch activado, y el progreso de cada turno de chat. Lo que llega solo necesita una transición que se note sin sobresaltar.

## 6. Pantallas

Cada pantalla necesita sus estados: cargando, vacío, error, sin conexión.

### 6.1 Acceso

- **Login**: una contraseña. Sin registro, sin "olvidé mi contraseña" (no existe recuperación: un solo usuario). Deliberadamente austero — es la puerta de un sistema privado.
- Error de contraseña: sin animación de "sacudida"; el campo conserva el foco.
- **Límite de intentos**: 5 intentos cada 15 minutos. Diseña el estado "demasiados intentos, espera X min".
- **Sesión**: dura 7 días; al vencer vuelve al login. Si vence en medio de una acción (ej. aprobando), tiene que quedar claro que *esa acción no se ejecutó*.
- **Face ID** (nuevo en iOS): propón su rol. Opciones: desbloquear la app al abrirla, y/o reautenticar antes de aprobar un dual-confirm. Nunca debe reemplazar la contraseña del servidor, solo proteger el acceso local.
- **Servidor**: la app apunta a `jin.jeanfranck.com` por defecto; un campo avanzado para cambiarlo (desarrollo local). Escondido, no protagonista.

### 6.2 Inicio

Responde "¿necesitan algo de mí?" en menos de un segundo:

- **Aprobaciones pendientes** — lo más prominente cuando hay alguna. Número, desglose (ej. "2 confirm · 1 dual-confirm") y cuánto le falta a la más urgente para escalar. **Sin pendientes, este bloque debe ceder protagonismo** en vez de mostrar un "0" grande y rojo: "nada esperando por ti" debe sentirse bien, no vacío.
- **Gasto de hoy** — dólares usados / límite del día, barra con umbrales 80% y 100%.
- **Modo de autonomía** actual y **kill switch** (si están en estado no-normal).
- **Actividad reciente** — últimas 5 acciones de los agentes (hora · herramienta).
- Sugerencias opcionales: turno de chat en curso, runs de orquestación activos, apps corriendo.

### 6.3 Aprobar — **la pantalla más importante**

Lista viva de acciones esperando decisión. Llegan solas, en tiempo real.

**Cada tarjeta muestra:**

- **Nivel de riesgo** (sección 4) y **qué herramienta** (`sendEmail`, `runCode`…).
- **Resumen del plan** en lenguaje natural ("Responder a Prof. Martínez confirmando asistencia").
- **El payload real, legible y visible por defecto** — el correo completo (destinatario, asunto, cuerpo), el evento exacto, el código que se va a correr. Es un objeto con campos arbitrarios según la herramienta (puede ser largo). Plegable para pantallas chicas, pero **abierto por defecto**: sin esto el usuario aprueba a ciegas.
- **"Influido por"** — qué fuentes externas leyó el agente antes de proponerlo (ej. `readEmails (1)`). Destacado como advertencia cuando existe.
- **Quién lo pidió** — el agente o sub-agente, o `web-chat` si salió de una conversación.
- **Cuánto falta para que escale/expire** — a las 12 h sin respuesta se escala; a las 24 h se descarta. El timeout **nunca aprueba solo**.

**Estados de la tarjeta:**

- pendiente (confirm) → Aprobar / Rechazar
- dual-confirm, primer paso → "Aprobar (1/2)"
- dual-confirm, **esperando los 30 s** → el botón muestra la espera ("Espera 22 s") y un texto tipo "la espera es deliberada: relee el payload". Al terminar → "Confirmar (2/2)"
- **ejecutándose** (ya aprobada, corriendo) → sin botones; "si dura más de 15 min, revisa el audit"
- **falló al ejecutarse** → "La última aprobación NO se ejecutó: <motivo>. No se reintenta sola." Nunca dejar al usuario creyendo que salió.
- error al aprobar/rechazar (ej. se confirmó demasiado pronto, sin conexión, sesión vencida) → visible en la tarjeta
- aprobada / rechazada (confirmación breve antes de desaparecer de la lista)

**Reglas duras de interacción:**

- **Nada de deslizar para aprobar/rechazar.** Un gesto accidental no puede ejecutar una acción con consecuencias.
- **Nunca aprobar sin conexión** (sección 7).
- Rechazar tiene que ser tan fácil como aprobar — no escondido.
- Diseña: el estado vacío ("nada esperando por ti"), 1 tarjeta, y 15 tarjetas acumuladas.
- Propón si la lista abre un **detalle a pantalla completa** por tarjeta (más espacio para payloads largos como código) o si todo vive en la tarjeta.

### 6.4 Chat

Conversación con el orquestador. El usuario pide **objetivos**, no tareas ("revisa mi correo y arma mi agenda de mañana") y el agente trabaja.

**Ahora el agente transmite su trabajo en vivo, en este orden dentro de un mismo turno:**

1. **Plan** — el agente declara sus pasos y los va marcando: pendiente, en curso, hecho, fallido (con nota opcional). El plan puede revisarse a mitad de camino (eso es su autocorrección).
2. **Herramientas en uso** — cada tool que llama aparece al empezar (nombre de la tool) y se actualiza al terminar: **éxito**, **error** (con motivo) o **diferida** (quedó esperando tu aprobación).
3. **Respuesta escribiéndose letra por letra** (streaming). Puede haber texto intermedio que después se reemplaza cuando el agente sigue trabajando — el diseño tiene que tolerar que el texto "en vivo" cambie.
4. **Respuesta final** + **qué modelo respondió** (ej. `claude-sonnet-5`; si el principal falló y entró el de respaldo, aparecen los dos).

**Casos del turno a diseñar:**

- Pensando (antes de que llegue nada).
- Turno largo con plan de 5 pasos y 3 tools.
- **Acción diferida**: el turno terminó pero dejó aprobaciones pendientes → acceso directo a esa tarjeta en Aprobar.
- **Rechazo del proveedor**: "No pude generar una respuesta: el modelo cortó la respuesta (su filtro de seguridad, no un error de Jin). Reformula o divide el pedido." — distinto de un error.
- **Se cayó a mitad de camino**: el texto parcial queda visible + aviso de error debajo. Nunca borrar lo que ya se mostró. No se reintenta solo (pudo haber ejecutado tools con efectos reales): el usuario decide.
- **Se perdió la conexión** antes de la respuesta: "no sabemos si el turno se completó — revisa Aprobar y Audit antes de reintentar".
- Límite de iteraciones alcanzado sin respuesta final.

**Conversaciones:**

- La conversación **persiste** (cambiar de pestaña, cerrar la app, volver) y Jin **recibe el historial** para tener contexto. Cuando el historial crece, el servidor lo comprime; el usuario no necesita ver eso.
- Las conversaciones viven **en este teléfono** — no se sincronizan con la web ni con Telegram. Propón cómo comunicarlo sin alarmar.
- Propón si hay una sola conversación continua con "nueva conversación", o una lista de conversaciones.
- Sugerencias de objetivos para el estado vacío (ej. "Revisa mi correo y arma mi agenda de mañana", "Qué entregas tengo esta semana").
- Mientras un turno corre, no se puede mandar otro.

### 6.5 Gasto y kill switch

- **Hoy**: dólares usados / límite, tokens usados / límite, porcentaje, umbrales **80%** (advertencia) y **100%** (bloqueo).
- **Última hora**: tokens y costo estimado de la última hora vs. la hora típica ("2,3× lo normal").
- **Kill switch**: si el consumo se desboca (más de 2× lo normal en una hora), el sistema **congela a todos los agentes solo**. Mostrar cuándo se activó y por qué.
- **Reanudar** exige **mantener pulsado 3 segundos** — fricción deliberada con progreso visible. Solo el humano puede iniciarlo; ningún agente puede pedirlo. Diseña el "soltaste antes de tiempo" y el error si no se pudo reanudar.
- *Deseado por el dueño, requiere trabajo en el servidor:* consumo por turno de chat y por modelo (ver notas de implementación). Diseña el espacio.

### 6.6 Autonomía

Controla cuánto actúa el agente sin pedir permiso. Tres modos:

| Modo | Qué hace |
|---|---|
| **Supervisado** (default) | Todo lo que pide aprobación la sigue pidiendo. |
| **Semiautomático** | Lo que pedía 1 aprobación se ejecuta solo y avisa, salvo una lista de **acciones protegidas** que siguen pidiendo permiso (ej. enviar correo). Lo irreversible sigue pidiendo 2. |
| **Automático** | Todo lo que pedía 1 aprobación se ejecuta solo y avisa. Solo lo dual-confirm sigue pidiendo 2. |

- Los modos relajados **caducan**: duración elegible con default y máximo por modo (hoy: semiautomático 24 h / máx. 72 h; automático 4 h / máx. 24 h). Cuenta regresiva hasta volver solo a supervisado.
- **Bajar la protección no se activa al tocar**: crea una aprobación **dual-confirm** que el usuario tiene que confirmar dos veces (30 s). Diseña el flujo: elegir modo → elegir duración → "esto NO lo activa todavía, crea una aprobación" → ir a Aprobar.
- **Volver a supervisado es inmediato**, un toque, siempre visible cuando está relajado.
- **Freno de emergencia**: si se autoejecutan más de 20 acciones en 1 h, vuelve solo a supervisado y avisa. Mostrar ese límite.
- Lista de acciones que siguen protegidas en semiautomático.

### 6.7 Claude Code

Canal de mensajería con una sesión de **Claude Code** (un agente de programación) que corre en el servidor del dueño. Es mensajería pura: **no aprueba ni ejecuta nada**.

- Mensajes de Claude Code al dueño y respuestas del dueño, estilo conversación, con hora.
- Mensajes con **opciones** ("¿Sigo con el deploy? [Sí] [No] [Esperar]") → botones de respuesta rápida; una vez respondido, quedan inactivos.
- **Responder a un mensaje específico** (citándolo) o escribir libre.
- El texto puede traer formato (negritas, código).
- Llegan en tiempo real; hoy también llegan por Telegram.
- *Deseado por el dueño, todavía no existe en el servidor:* ver con qué **modelo** trabaja esa sesión (Haiku / Sonnet / Opus), su **nivel de esfuerzo** (bajo/medio/alto), **tokens** consumidos, estado de la sesión (activa / login vencido), y **cambiar modelo y esfuerzo** desde la app. Diseña este panel como parte de la pantalla, marcado como "próximamente" si hace falta.

### 6.8 Board de orquestación

Cuando un objetivo es complejo, el orquestador lo parte en **tickets** y los reparte entre varios sub-agentes que trabajan en paralelo.

- **Lista de runs** (objetivos): objetivo, estado (en curso / bloqueado / hecho / fallido / detenido), fecha. Vacío: "los objetivos se piden en el Chat, no acá" → acceso al chat.
- **Detalle de un run**: tickets agrupados por estado (pendiente / en curso / hecho / fallido / bloqueado); cada ticket con su sub-agente asignado, descripción, dependencias de otros tickets, herramientas permitidas, resultado, y un **hilo de comentarios** (autor: orquestador / sub-agente / dueño; tipo: nota / resultado / conflicto / resolución).
- **Conflictos entre agentes** se muestran, no se esconden: cuando dos sub-agentes se contradicen y el orquestador no pudo resolverlo, escala al humano → destacado arriba, con acceso a su aprobación.
- En desktop era un tablero tipo Jira; en iPhone repiénsalo (secciones por estado, carrusel de columnas, filtros…).

### 6.9 Apps

Los agentes pueden levantar **aplicaciones reales** (un frontend de prueba) accesibles por una URL pública temporal (`<slug-aleatorio>.jinserver.com`).

- Lista: URL, estado (corriendo / expirada), tiempo de vida restante (máximo 24 h, expiran solas).
- **Abrir**: el contenido lo generó una IA y **no es parte confiable del sistema** — el paso de "Jin" a "esa app" tiene que marcarse claramente (etiqueta "GENERADO POR IA · AISLADO · NO CONFIABLE" o equivalente).
- **Detener** (con confirmación, es destructivo).

### 6.10 Memoria

Buscador sobre la memoria de largo plazo del agente: *lo que Jin cree saber de ti, y de dónde lo sacó*.

- Búsqueda semántica en lenguaje natural ("cómo prefiero que me agenden las asesorías").
- Resultado: el recuerdo textual, su **tipo** (hecho / preferencia / lección / episodio), su **fuente** y **fecha**, y qué tan parecido es a la búsqueda.
- Filtro opcional por tipo.
- Vacío: "ningún recuerdo pasa el umbral de similitud".

### 6.11 Audit

Registro **inmutable y encadenado criptográficamente** de cada acción del sistema. Es forense.

- Lista paginada (más reciente primero): momento, tipo de acción, herramienta, actor, estado de aprobación.
- Detalle por entrada: resumen del plan, quién aprobó, **inputs externos que influyeron**, hash de la entrada y hash anterior (la cadena).
- Monoespaciado donde corresponde. En iPhone: lista escaneable + detalle, no una tabla ancha.

### 6.12 Ajustes

Mínimos: servidor, Face ID, notificaciones (ver 8), cerrar sesión, versión. Nada más — casi todo se configura en el servidor.

### Fuera de la app

- **Editor de código** con comentarios de IA: existe en la web, es solo de escritorio. No se diseña para iPhone (como mucho, una fila en "Más" que diga "solo en escritorio").

## 7. Conectividad

4 estados de conexión, distintos entre sí (no solo por color — forma también):

| Estado | Significado | Qué se puede hacer |
|---|---|---|
| **En vivo** | Tiempo real funcionando | Todo |
| **Reconectando** | Hay red, se perdió el tiempo real hace poco | Todo; la lista se refresca a mano |
| **Sin tiempo real** | Lleva >15 s reconectando | Todo, pero la lista puede estar incompleta — decirlo |
| **Sin conexión** | Sin red | Solo lectura de lo último sincronizado, con su hora |

**Regla de seguridad: nunca aprobar, rechazar, reanudar el kill switch ni cambiar el modo de autonomía sin conexión.** Nada queda en cola: una aprobación que se sincroniza 3 horas tarde es un problema de seguridad, no una comodidad. Diseña ese bloqueo explícitamente (botones deshabilitados con el porqué).

Al volver de segundo plano, la app se resincroniza: diseña el momento "actualizando" sin que los datos viejos parezcan actuales.

## 8. Capacidades nativas de iOS

Propón cómo usar cada una. Las marcadas **(requiere servidor)** no existen todavía — diséñalas igual, son el reemplazo natural del bot de Telegram:

- **Notificaciones push (requiere servidor)**: aprobación nueva, aprobación por escalar/expirar, acción `notify` ejecutada, alerta de presupuesto 80%/100%, kill switch activado, mensaje de Claude Code, resumen matutino (hoy Telegram manda a las 06:00 las entregas de Canvas del día), turno de chat terminado. Diseña la pantalla de permisos y los ajustes por tipo.
  - **Regla dura: nunca aprobar desde la notificación ni desde la pantalla de bloqueo.** La notificación lleva a la tarjeta; el payload se lee antes de decidir.
  - Las respuestas rápidas a Claude Code sí podrían ir en la notificación (es mensajería, no ejecuta nada) — propón.
- **Live Activity / Dynamic Island (requiere servidor para actualizarse en segundo plano)**: candidatos — cuenta regresiva del dual-confirm, turno de chat largo en curso, modo de autonomía relajado con su cuenta regresiva, kill switch activo.
- **Widgets (pantalla de inicio / bloqueo)**: aprobaciones pendientes, gasto de hoy, modo de autonomía. Solo lectura; tocar abre la app.
- **Háptica**: al llegar una aprobación, al aprobar/rechazar, al completar el hold de 3 s, al terminar la espera de 30 s.
- **Face ID**: ver 6.1.
- Dynamic Type y VoiceOver completos: la tarjeta de aprobación tiene que ser usable con texto grande.

## 9. Dirección visual

- **No partas del diseño web actual** (glassmorphism oscuro con acentos rojo-naranja): el dueño no quiere conservarlo. Propón una identidad nueva, pensada para iOS.
- Tiene que sentirse **nativa de iOS actual** (componentes, gestos y tipografía del sistema cuando sirvan) pero con carácter propio — no una app genérica de plantilla.
- **Oscuro por defecto**, claro funcional. Se usa mucho de noche.
- **Densa pero respirable**: es una cabina, no una landing. Sin ilustraciones decorativas ni héroes vacíos.
- **Monoespaciado** para nombres de herramientas, payloads, hashes, identificadores, modelos. Proporcional para el resto.
- El sistema de riesgo (sección 4) manda sobre la paleta: el color de marca no puede competir con el rojo de "irreversible".
- **El nombre es Jin.** Propón ícono de app y pantalla de arranque.

## 10. Qué NO diseñar

- Onboarding, tours, tutoriales, pantallas de bienvenida
- Registro, recuperación de contraseña, perfiles, roles, multi-cuenta
- Landing, pricing, marketing
- Configuración extensa
- Editor de código
- iPad o Mac (solo iPhone por ahora)

## 11. Entregable

1. **Fundamentos**: color (incluido el sistema de 4 niveles de riesgo), tipografía, espaciado, íconos, ícono de app.
2. **Componentes**: tarjeta de aprobación en sus 4 niveles y todos sus estados (6.3); plan en vivo; fila de tool en curso; burbuja de respuesta en streaming con su modelo; estado de rechazo del proveedor; medidor de presupuesto; botón de mantener pulsado; selector de modo de autonomía; banners globales apilados; indicador de conexión en sus 4 estados; mensaje de Claude Code con opciones; ticket del board; fila de audit; notificación push.
3. **Pantallas**: acceso, inicio (con y sin pendientes), aprobar (vacío / 1 / 15), chat (turno en vivo y terminado), gasto (normal / kill switch), autonomía, Claude Code, board y detalle de run, apps, memoria, audit y detalle, ajustes, "Más".
4. **Estados** de carga, vacío, error y sin conexión por pantalla.

Empieza por la **tarjeta de aprobación en sus 4 niveles y con la espera del dual-confirm**. Si ese componente funciona en un iPhone con una mano, el resto se ordena detrás de él.

═══════════════════════════════════════════════════════════

## Notas para implementación (no pegar en Claude Design)

Brechas entre lo que el brief pide y lo que el servidor hace hoy — a resolver en Jin_Core antes o durante la construcción de la app:

| Pide el brief | Estado real | Trabajo |
|---|---|---|
| Autenticación de la app | **Listo.** `POST /api/auth/login` devuelve `accessToken` en el body; HTTP acepta `Authorization: Bearer`; WS acepta `auth.token`. Mismo esquema que Jin_CLI. | Ninguno. Token en Keychain. |
| Streaming del chat (6.4) | En Jin_Core#50 + #51 y Jin_Web#18 (en revisión). Evento `chat:progress` en el namespace `/chat`. | Mergear. |
| Modelo que respondió (6.4) | `modelsUsed` viene en el `chat:response` por WS (no en `POST /api/chat`). | Ninguno para la app. |
| Historial del chat (6.4) | `ChatDto` ya acepta `history`; el turno devuelve `compactedHistory` cuando comprime. La web **no manda historial** (por eso Jin "no tenía contexto"). | Solo del lado del cliente. Vale la pena arreglarlo también en Jin_Web. |
| Push notifications, Live Activities remotas (8) | No existe. Hoy todo sale por Telegram (`TelegramBotService`, eventos `HITL_*`, `MORNING_ALERT_EVENT`, `RELAY_MESSAGE_CREATED_EVENT`). | Registro de device token + envío APNs como un canal más escuchando los mismos eventos. Requiere cuenta de Apple Developer. |
| Tokens/costo por turno y por modelo (6.5) | El budget registra uso por sesión pero no lo expone por turno. | Agregar uso al `AgentTurnResult` y/o endpoint de desglose. |
| Panel de modelo/esfuerzo/tokens de Claude Code (6.7) | No existe: el puente es solo mensajería (`/api/bridge/*`). | Requiere que la sesión de la VM reporte su estado y acepte cambios (Remote Control u otro mecanismo). |
| Filtros en Audit (6.11) | La API solo pagina (`limit`, `cursor`). | Filtros opcionales en `GET /api/audit`. |
| Comentar como dueño en un ticket (6.8) | Solo lectura para el dueño. | Solo si el diseño lo propone. |
| Widgets (8) | Viable con los endpoints actuales (lectura con el token). | Solo cliente (App Group + Keychain compartido). |

Entorno: esta Mac no tiene Xcode instalado (solo Command Line Tools) — hay que instalarlo antes de construir la app.
