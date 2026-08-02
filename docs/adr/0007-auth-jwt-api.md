# ADR 0007 — Auth JWT single-user + API REST/WebSocket

## Contexto

BLUEPRINT §4.1/§5.2 y AGENTS.md §5 exigen que Jin_Core exponga una API auth-protegida como frontera de seguridad para el Web Dashboard (Fase 6.2) y la CLI (Fase 6.4) — hoy todo pasa exclusivamente por Telegram.

## Decisiones

### 1. JWT vía cookie `__Host-` para Web, Bearer token para CLI

Mismo mecanismo (JWT firmado, `JWT_SECRET` fail-fast), dos formas de entrega según el cliente. La cookie `__Host-jin_session` (httpOnly, secure, sameSite=strict, sin `Domain`) es la que BLUEPRINT 5.2 exige específicamente para que el navegador nunca la adjunte a peticiones cross-site — incluida cualquier app generada por un agente bajo `jinserver.com`. La CLI no tiene ese vector de ataque (no es un navegador), así que recibe el token en el body de login y lo guarda localmente.

### 2. Sin blocklist de logout — límite conocido, aceptado

Invalidar un JWT stateless antes de su expiry natural requiere una blocklist (Redis, ya disponible) — se decide NO construirla en v1: logout solo borra la cookie cliente. Expiry corto (7 días) acota el riesgo. Si se captura un token, sigue siendo válido hasta expirar. Aceptable para un sistema single-user sin superficie de robo de sesión más allá de que el propio owner comprometa su dispositivo — mismo nivel de riesgo que hoy tiene el bot de Telegram (`TELEGRAM_OWNER_CHAT_ID` es la única barrera, sin rotación).

### 3. `JinErrorFilter` global — prerequisito encontrado durante la investigación, no planeado originalmente

`JinError` extiende `Error`, no `HttpException` de Nest — sin un filtro global, cualquier excepción de dominio (`PendingApprovalNotFoundError`, `SecondApprovalTooEarlyError`, etc.) que llegue a un controller devolvería 500 en vez de su `httpStatus` real. Ningún controller existente en Jin_Core lo necesitaba (Telegram atrapa cada error a mano, nunca lo propaga a una respuesta HTTP) — los endpoints REST nuevos sí, así que se construye acá.

### 4. Rate limiting con Redis real, ventana fija (no sliding window)

Redis ya está desplegado en Jin_Infra (Postgres del mismo modo, StatefulSet con backup) pero Jin_Core nunca lo usó — ni BullMQ (descartado explícitamente, ver STATUS.md) ni nada más lo toca. Conectar por primera vez es cero infraestructura nueva. BLUEPRINT 9.7 pide "sliding window"; se implementa ventana fija (`INCR`+`EXPIRE` atómico) por ser sustancialmente más simple de tener correcta sin condiciones de carrera, y la diferencia práctica entre fija y deslizante es irrelevante para un sistema de un solo usuario sin adversarios multi-tenant reales — el objetivo es frenar fuerza bruta contra `/login`, no modelar tráfico a escala.

### 5. WebSocket con eventos híbridos: uno genuinamente push, el resto polling disfrazado de push

`pending-approval:new` usa `@nestjs/event-emitter` con un único emit point real (`DualConfirmService.createPendingApproval()`) — es el único evento donde la latencia de "hasta 5 minutos" sería un problema real de UX (una aprobación nueva debería aparecer ya). `budget:alert`/`kill-switch:activated` replican el mismo sondeo por `@Cron` que ya usa `TelegramBotService.checkBudgetAlerts()` (mismo dedup por umbral/día) — no se extrae esa lógica a un servicio compartido todavía, siguiendo AGENTS.md 1.1 ("esperar a ver el patrón 3 veces antes de abstraer"): con Telegram + este gateway son 2 consumidores, no 3.

### 6. Controllers viven dentro de cada módulo de dominio, no en un módulo "api" genérico

Sigue AGENTS.md 4.2 ("un módulo por dominio de negocio") y el precedente ya establecido en Jin_Executor (`execute.controller.ts` dentro de `src/execute/`, junto al servicio que envuelve). `DualConfirmService`/`AuditService` ganan métodos de lectura nuevos (`listPending()`/`listRecent()`) en vez de que el controller arme el query — controllers sin lógica de negocio (AGENTS.md 4.3), mismo motivo por el que ya se centralizó `ApprovalExecutionService` en la Fase 4.2 en vez de dejar que Telegram orquestara aprobar+auditar a mano.

### 7. `POST /api/chat` sin persistencia de sesión server-side

A diferencia de Telegram (que persiste el transcript en Postgres porque su UX de comandos/reconexión lo exige — Fase 5.3), el chat de la API REST no tiene ese requisito explícito todavía. El caller gestiona su propio historial y lo manda en cada request. Decisión deliberadamente mínima: agregar persistencia después es un cambio aditivo al contrato, no rompe nada existente.

### 8. Argon2id para el hash de la contraseña del owner

Recomendación OWASP vigente sobre bcrypt (mejor resistencia a ataques por GPU/ASIC). Costo de dependencia nueva idéntico a bcrypt. Hash precomputado una vez por el owner vía `scripts/hash-password.ts`, pegado como `OWNER_PASSWORD_HASH` — nunca la contraseña en claro en ningún lado (env, código, logs).

## Consecuencias

- Swagger UI se mueve de `/api` a `/docs` — libera el path que BLUEPRINT 5.2 reserva para la API de negocio real.
- `JinErrorFilter` queda disponible para TODO el repo, no solo los endpoints de esta fase — cierra un gap real preexistente (controllers futuros heredan el comportamiento correcto sin tener que descubrirlo de nuevo).
- Primera vez que Jin_Core habla con Redis — Postgres y Redis quedan como las dos dependencias de infraestructura reales del repo (BullMQ sigue explícitamente fuera).
- El JWT sin blocklist es una superficie de riesgo aceptada y documentada, no un descuido — revisar si Fase 7.3 (hardening) amerita agregar la blocklist con lo que ya deja armado este PR (Redis ya conectado).

## Alternativas consideradas

- **Blocklist de logout desde el día 1:** descartado para v1 — complejidad extra (TTL igual al del JWT, chequeo en cada request) para un riesgo bajo en un sistema single-user; queda anotado para Fase 7.3 si hace falta.
- **Sliding window real (Lua script atómico en Redis) para rate limiting:** descartado — correcto pero mucho más código propio a mantener sin ganancia práctica para este caso de uso.
- **Socket.IO para todo, incluidos `budget:alert`/`kill-switch:activated` vía push real (EventEmitter en vez de polling):** descartado por ahora — requeriría mover la lógica de umbrales/dedup fuera de `TelegramBotService` a un servicio compartido antes de tener 2 consumidores reales de esa lógica; se prefiere duplicar el sondeo (barato, ya probado) a abstraer prematuramente (AGENTS.md 1.1).
- **Módulo "api" único agrupando todos los controllers nuevos:** descartado — rompe la convención ya establecida de un módulo por dominio (AGENTS.md 4.2) y precedente de Jin_Executor.
