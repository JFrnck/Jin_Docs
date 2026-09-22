# ADR 0012 — Puente Claude Code ↔ owner por un bot de Telegram separado

## Contexto

El owner quiere levantar **Claude Code dentro de la VM** y que esa sesión le escriba por Telegram: avisar cuando termina algo, reportar un problema y **preguntarle con botones de opciones**, sin tener que estar delante del Mac.

Antes de esto no existía ninguna vía. El bot de Jin (`@Jin_dev_bot`) solo habla con `ownerChatId` y su único emisor es `TelegramBotService`.

**El riesgo que define el diseño:** el chat de Jin es el canal de aprobaciones HITL (regla de oro #7). Claude Code lee logs, issues y páginas web todo el día — es decir, ingiere contenido no confiable de forma continua. Si pudiera escribir en el chat de Jin, una **inyección de prompt** podría producir un mensaje con aspecto de Jin que empuje al owner a aprobar algo. El atacante no necesitaría romper HITL: le bastaría con hablar por el canal donde el owner ya confía.

Aprobado por el owner el 2026-09-21.

## Decisiones

### Un segundo bot, no el de Jin

Dos chats separados. El del puente **no tiene maquinaria de aprobación**, así que la confusión no es improbable: es imposible.

Esto no se sostiene con una convención. El módulo `src/relay/` **no inyecta** `ApprovalExecutionService` ni `DualConfirmService`, y `relay.isolation.spec.ts` falla si alguien importa `hitl/`, `hitl-policy/`, `audit/`, `agent/`, `executor-client/`, `integrations/` o `tools/` desde ahí. Resolver una aprobación de Jin desde el puente no es "algo que no hacemos": es algo que no se puede hacer sin que un test lo diga en voz alta — que es justo el momento en que hay que discutirlo.

El test lee el código **con los comentarios quitados**. La primera versión falló porque el comentario del módulo menciona `HitlModule` para explicar que no lo importa. Un test de seguridad que salta por una palabra en un comentario entrena a ignorarlo, y un test de seguridad ignorado es peor que no tenerlo.

### La frontera que no se cruza: Jin nunca lanza Claude Code

El puente es de una sola dirección **en cuanto a privilegio**. Claude puede hablarle al owner a través de la infraestructura de Jin; **el agente de Jin no puede lanzar ni controlar Claude Code**.

Lo contrario convertiría cualquier inyección de prompt en Jin en **ejecución arbitraria con los permisos de Claude sobre la VM** — que son altos por diseño. Jin está construido entero alrededor de la idea de que el agente no ejecuta nada sin pasar por HITL y el Executor; darle un intérprete de propósito general por la puerta de atrás anularía esa arquitectura de un plumazo.

Si algún día hace falta lo contrario (que Jin dispare una tarea de Claude), no se hace ampliando este puente: se diseña aparte, con su propio ADR.

### Long polling, no webhook

El bot de Jin usa webhook porque ya tiene ingress público. Este usa **long polling**: no abre ninguna ruta pública nueva, solo una conexión saliente a `api.telegram.org`. Menos superficie que defender por una función opcional.

Con backoff exponencial: un 409 (`terminated by other getUpdates`) es esperable durante un rollout, mientras el pod viejo no ha muerto. Y con `onModuleDestroy`, para que el pod viejo deje de competir por el bot en vez de reintentar para siempre.

### Credencial propia, no el JWT del owner

`RELAY_TOKEN`, comparado con `timingSafeEqual`. Son credenciales distintas a propósito y en las dos direcciones: **el token del puente no abre el resto de la API de Jin, y el JWT del owner no abre el puente.** Ambas mitades tienen su test e2e.

Un `RELAY_TOKEN` filtrado solo puede relayar mensajes: ni aprobar, ni leer datos de Jin, ni chatear como el owner.

`/api/relay` además se bloquea en el ingress público; el CLI llega a `jin-core:3000` desde el nodo.

### Tabla propia, fuera de la cadena de audit

`relay_messages` (migración 0013). **No** escribe en `audit_log`: esa cadena está hash-encadenada y existe para acciones HITL. Mensajería no es una tool call, y meterla ahí diluiría el significado de la cadena — que sirve precisamente porque todo lo que contiene es del mismo tipo.

`inbox` marca consumido en el **mismo** `UPDATE…RETURNING` que lee. Con un SELECT y un UPDATE aparte, dos lecturas concurrentes (dos sesiones, o un reintento) harían que Claude procesara dos veces la misma instrucción del owner.

### Configuración opcional, no requerida

**Desviación deliberada del plan aprobado.** El plan pedía meter `TELEGRAM_RELAY_BOT_TOKEN` y `RELAY_TOKEN` en `REQUIRED_SECRET_KEYS`, y advertía del riesgo.

Son **opcionales**. Requerirlas dejaría el pod en CrashLoopBackOff por una función nueva que ni siquiera es del núcleo — exactamente lo que pasó con `INFISICAL_SITE_URL` en el primer despliegue real. Sin ellas el puente queda apagado y Jin arranca igual.

Un `refine` exige que vayan **las dos juntas o ninguna**: media configuración es peor que ninguna (con bot pero sin token de API el puente escucha y nadie puede hablarle; al revés, el CLI acepta mensajes que no llegan a ningún lado).

### Límite de 30 mensajes/hora

Un Claude en bucle no puede inundar el móvil del owner. Al pasarse, el endpoint responde 429 y no manda nada.

## Consecuencias

- Migración 0013 (retrocompatible, tabla nueva).
- Un segundo bot que el owner debe crear en BotFather y saludar una vez — un bot no puede escribir primero.
- Dos claves nuevas en Infisical y en el rol `jin-core-reader` (verificándolo **en la base**: la UI de v0.99 falla en silencio).
- Los e2e arrancan con el puente **encendido** a propósito. Apagado, el guard responde 503 a todo y los tests de autenticación pasarían sin comprobar nada — el mismo error que ya cometimos una vez con el probe de aislamiento del Executor, que reportaba `BLOCKED:` ante cualquier excepción.
- `/api/relay/*` entra en `contracts/openapi.json`. No lo consumen Web ni CLI.

## Alternativas consideradas

- **Usar el bot de Jin con un prefijo tipo "[Claude]":** rechazado. Un prefijo es una convención visual, y las convenciones visuales son exactamente lo que una inyección de prompt sabe imitar.
- **Webhook en vez de long polling:** rechazado. Abriría una ruta pública nueva para una función opcional.
- **Escribir en `audit_log`:** rechazado. Rompe la homogeneidad que le da sentido a la cadena.
- **Claude Code Remote Control:** no es una alternativa, es una función distinta — permite manejar *esta* sesión desde el móvil, sin código y sin pasar por Jin. No sirve para el caso que motiva el puente: una sesión autónoma corriendo sola en la VM que necesita reportar. Se le mencionó al owner para que la decisión fuera informada.
- **Que Jin pueda lanzar Claude Code:** rechazado, y no por ahora sino por diseño. Ver arriba.
