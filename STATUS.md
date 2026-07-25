# STATUS

## Última actualización: 2026-07-25 (America/Lima) — actualización 7

> **Antigravity ya está activo** (ver abajo, Fase 3.1/2.4/4.2). Retoma ownership normal de `docs/WORKFLOW.md` sección 2 — Claude Code ya no asume tareas `[ANTIGRAVITY]` por defecto, salvo negociación puntual vía esta misma nota.

## En progreso

### Claude Code

- **Repo:** ninguno activo ahora mismo.
- **Descripción:** Fase 4.1 (Budget guard + kill switch) completa — [PR #6](https://github.com/JFrnck/Yormun_Core/pull/6), **mergeado**. Prerequisito "ejecutar al aprobar" completo — [PR #8](https://github.com/JFrnck/Yormun_Core/pull/8), **mergeado**. Revisión de 3 rondas + merge de [PR #9](https://github.com/JFrnck/Yormun_Core/pull/9) (Fase 4.2, Antigravity). Fase 4.3 (Memoria extendida sqlite-vec) completa — [PR #10](https://github.com/JFrnck/Yormun_Core/pull/10), **mergeado**. Ver detalle abajo.
- **Próximo:** ninguno pendiente en el roadmap actual — Fase 2, 3.1, 4.1, 4.2 y 4.3 cerradas. A la espera de que el owner defina Fase 5+.

### Antigravity

- **Repo:** Yormun_Core
- **Rama:** `feature/antigravity/google-workspace` — [PR #9](https://github.com/JFrnck/Yormun_Core/pull/9), **mergeado a `main`, rama borrada**.
- **Descripción:** Fase 4.2 completa — Google Calendar + Gmail (`src/integrations/google/`). **Fin de Fase 4.2.**
- **Estado:** 🟢 Sin tarea activa — a la espera de la siguiente fase que le asigne el owner (no hay ninguna en el roadmap actual asignada a Antigravity; Fase 4.3 es de Claude Code).
- **Archivos creados/modificados:** `src/integrations/google/` (`GoogleOAuthService` con `getDaysSinceLastRefresh()`, Calendar client/tools, Gmail client/tools con sanitización anti-CRLF en `to`/`subject`), migración `drizzle/0003_google_oauth_state.sql` + `.down.sql`, `GoogleModule` registrando executors en `ToolExecutorRegistry`, comando `/google-oauth-refreshed` + alerta proactiva a Telegram cuando `days >= 6` (con dedupe diario). 193 tests unitarios + 40 de integración en Postgres real + e2e, CI GitHub Actions verde — **verificado independientemente por Claude Code en 3 rondas de revisión** (ver sección de feedback abajo), no solo el self-report.


## Feedback Ronda 3 (Telegram) para Antigravity, enviado 2026-07-23

Claude Code verificó de forma independiente los 3 puntos de la Ronda 2. 2 resueltos genuinamente; 1 quedó a medias:

- ✅ **Tipos en specs**: `tsc --noEmit` da 0 errores. Resuelto.
- ✅ **`botInfo` hardcodeado**: sacado del constructor; `onModuleInit()` llama `await this.bot.init()` real, valida el token contra Telegram al arrancar. Resuelto.
- ⚠️ **Webhook `secret_token`: resuelto solo a medias.** `TELEGRAM_WEBHOOK_SECRET` quedó como `z.string().min(1).optional()` en `env.schema.ts`, y `validateWebhookSecret()` devuelve `true` (permite) si `this.webhookSecret` no está configurado. Si en producción no se setea esa variable (nada lo exige, no falla al arrancar), el webhook vuelve a aceptar cualquier POST sin validar nada — la vulnerabilidad original, ahora condicionada a que el owner se acuerde de una variable que el sistema nunca reclama. Los tests cubren "secret configurado, header correcto/incorrecto/ausente" pero ninguno cubre "el secret no está configurado en absoluto" — el escenario más peligroso, sin probar. Dado que es el único canal de aprobación de todo el HITL, `TELEGRAM_WEBHOOK_SECRET` debe ser **requerida** (sacar `.optional()`), mismo patrón fail-fast que `TELEGRAM_BOT_TOKEN`.

## Feedback Ronda 2 (Telegram) para Antigravity, enviado 2026-07-23 (2 de 3 puntos resueltos, ver Ronda 3)

Claude Code revisó el código real de la rama (no solo el self-report), corriendo `pnpm lint`/`test`/`build` de forma independiente + `npx tsc --noEmit -p tsconfig.json` sobre todo el proyecto (`nest build` usa `tsconfig.build.json`, que excluye specs — por eso esto no se ve en `pnpm build`). 3 puntos:

1. **Gap de seguridad — el webhook no valida que la petición venga de Telegram.** `TelegramWebhookController` procesa cualquier POST a `/telegram/webhook`. La única barrera es `chat.id === ownerChatId` **dentro** del payload — pero el endpoint es público, así que cualquiera que sepa/adivine el chat_id del owner podría forjar un Update y el bot lo procesaría como si fuera él. Es el único canal real de aprobación del sistema HITL — hay que cerrarlo con `secret_token` en `setWebhook(url, { secret_token })` + validar el header `X-Telegram-Bot-Api-Secret-Token` en el controller antes de procesar (401 si no matchea).
2. **Errores de tipos reales en los specs, invisibles para `pnpm test`/`pnpm build`.** `nest build` excluye `**/*spec.ts` vía `tsconfig.build.json`, y `vitest` transpila con SWC sin type-check. `npx tsc --noEmit -p tsconfig.json` encontró: el `chat` fake en 3 tests le falta `first_name` (requerido por `PrivateChat`), `cmd` posiblemente `undefined` en `text.split(' ')[0]` (`noUncheckedIndexedAccess: true`), el transformer de `mockSendMessage` no matchea el tipo `ApiResponse` de grammY, y el `Update` fake del controller-spec le faltan `message_id`/`date`/`chat`. Ninguno rompe en runtime, pero violan el `strict: true` del repo — correr `tsc --noEmit` antes de dar un PR por cerrado.
3. **Menor — `botInfo` hardcodeado en el constructor de producción.** `new Bot(token, { botInfo: {...} })` en `TelegramBotService` (no solo en tests) evita que el bot valide el token real contra Telegram al arrancar (sin fail-fast) y deja `bot.botInfo`/`ctx.me` con una identidad falsa para siempre. Sacarlo, llamar `await this.bot.init()` real en `onModuleInit()`, y en tests hacer que el transformer también responda a `getMe`.

Lo que sí está sólido: `TELEGRAM_OWNER_CHAT_ID` como número (bien corregido en Ronda 1), `/budget` con stub honesto, consumo correcto de `DualConfirmService`/`AuditService`/`ModelRouterService`, `SELECT` directo sobre `pendingApprovals` sin tocar `src/hitl/**`, wiring de módulos correcto.

## Feedback Ronda 1 (Telegram) para Antigravity, enviado 2026-07-23 (ya resuelto)

1. **Bug real — `chat.id` es `number`, `TELEGRAM_OWNER_CHAT_ID` no puede validarse como `z.string()`.** En grammY, `ctx.chat.id` es `number`. Comparar `ctx.chat.id === TELEGRAM_OWNER_CHAT_ID` con el env var tipado como string da `false` siempre (`number === string`) — bloquearía al owner de su propio bot en silencio. Usar el mismo idioma que `PORT` en `env.schema.ts`: `TELEGRAM_OWNER_CHAT_ID: z.coerce.number().int()`, comparar como número directo.
2. **`/budget` no tiene nada real que reportar — Budget guard (Fase 4.1) no existe todavía.** `src/budget/**` no está construido. `/budget` debe ser un stub honesto ("Budget guard todavía no está implementado, llega en Fase 4.1"), no un número fabricado — mismo patrón que `CalendarNotImplementedError`.

Menor: aclarar quién registra el webhook con Telegram (`bot.api.setWebhook(url)`) — `main.ts` al arrancar, o paso manual del owner.

Lo que el plan sí acierta: consumo correcto de `DualConfirmService`/`AuditService`/`ModelRouterService` sin reimplementarlos, `SELECT` directo sobre `pendingApprovals` en vez de tocar `src/hitl/**`, auth por chat_id whitelisted, webhook vía controller NestJS, env vars fail-fast, tests con grammY mockeado.

## Fase 3.1 (completada, referencia)

`src/integrations/canvas/**` mergeado en PR #4. `CanvasClientService` (rate limit 30 req/min), `CanvasToolsService` (3 tools, ambas `auto` sanitizando con `wrapUntrustedContent`), `ShadowingService` (cron nocturno vía `ModelRouterService.complete('long_context', ...)`). Handoff pendiente: `canvasScheduleStudyBlock` lanza `CalendarNotImplementedError` (501) a la espera de Google Calendar (Fase 4.2).

## Feedback Ronda 2 para Antigravity — plan Fase 3.1 (Canvas), enviado 2026-07-23

Claude Code revisó el plan actualizado (ya consumiendo `wrapUntrustedContent`/`ModelRouterService` en vez de reimplementarlos) contra el código real de `src/model-provider/` y `src/security/` recién mergeado, y contra `AGENTS.md` §8.1. Ajustar antes de implementar:

1. **Bug bloqueante — la llamada a `ModelRouterService.complete()` no matchea la interfaz real.** El plan usa `complete('long_context', { prompt, systemPrompt })`. `ModelCompletionRequest` (`src/model-provider/model-provider.types.ts`) no tiene ningún campo `prompt`, y le faltan dos campos **requeridos** (`messages`, `maxOutputTokens`, `temperature`) — esto no compilaría. Forma correcta:
   ```ts
   await this.modelRouterService.complete('long_context', {
     systemPrompt: '...',
     messages: [{ role: 'user', content: wrappedContent }],
     maxOutputTokens: 8000, // = config/models.yaml → long_context.max_tokens_output
     temperature: 0.4,      // = config/models.yaml → long_context.temperature
   });
   ```
   El router no copia `maxOutputTokens`/`temperature` del profile automáticamente — es responsabilidad del caller pasarlos explícitos.
2. **El stub de `canvasScheduleStudyBlock` usa la excepción equivocada.** El plan dice seguir "el patrón `ModalService`", pero ese patrón (`Yormun_Executor/src/modal/errors.ts`) es `ModalNotImplementedError extends YormunError` — clase propia con `code`/`httpStatus`/`cause` — no `NotImplementedException` de `@nestjs/common`. `AGENTS.md` §8.1 exige que todo error tenga su clase custom extendiendo `YormunError`. Debe ser `CalendarNotImplementedError extends YormunError` (`code: 'CANVAS_CALENDAR_NOT_IMPLEMENTED'`, `httpStatus: 501`) en `src/integrations/canvas/errors.ts`.
3. **Falta sanitizar `canvasListAssignments`, no solo `canvasGetCourseContent`.** El plan envuelve con `wrapUntrustedContent` el resultado de `canvasGetCourseContent` pero no el de `canvasListAssignments` — y esa tool también es `hitlLevel: 'auto'` (su resultado vuelve directo al contexto del LLM como resultado de tool-call). Títulos/descripciones de tareas son contenido externo igual que el contenido de curso — `AGENTS.md` §5.1 exige envolver ambas.

Menor (no bloqueante): usar `.spec.ts` co-ubicado para los tests, no una carpeta `__tests__/` separada — es la convención del resto del repo (hitl, audit, model-provider).

Lo que el plan sí acierta en esta ronda: consumo correcto del sanitizer y el model-router sin reimplementarlos, reuso correcto de las 3 tools ya declaradas en `registry.ts`, `CANVAS_BASE_URL`/`CANVAS_API_TOKEN` requeridas con fail-fast, rate limit de 30 req/min, mocks HTTP en tests, y la referencia a "Fase 4.2" (`PROMPTS.md`) para Google Calendar es correcta.

## Feedback Ronda 1 para Antigravity — plan Fase 3.1 (Canvas), enviado 2026-07-23 (ya resuelto)

Claude Code revisó el plan propuesto (Canvas client + tools + shadowing cron) contra `AGENTS.md`, `docs/PROMPTS.md` §3.1, las golden rules de `docs/BLUEPRINT.md` §15 y el estado real del código en `Yormun_Core`. Ajustar antes de implementar:

1. **Sanitizador de injection incorrecto.** El plan propone un `canvas-sanitizer.ts` propio con tag genérico `<untrusted_content>`. `AGENTS.md` §5.1 (golden rule #6) exige una función **compartida** `wrapUntrustedContent(content, source, sessionNonce)` en `src/security/injection-sanitizer.ts`, con tag `<untrusted_content_{sessionNonce}>` (nonce por sesión + escape HTML) y `generateSessionNonce()`. Ese módulo **no existe todavía** (`src/security/` solo tiene `.gitkeep`). Además `Yormun_Docs/CLAUDE.md` §3 asigna `src/security/**` a Claude Code, no a Antigravity — coordinar antes de tocarlo, no reimplementar una versión propia y más débil dentro de `integrations/canvas/`.
2. **Falta la infraestructura de model routing.** `PROMPTS.md` §3.1 exige usar Gemini 3.1 Pro para el resumen del shadowing; golden rule #5 prohíbe modelos hardcoded — todo debe pasar por `model-provider/router.ts` + `config/models.yaml` (ver `docs/MODEL_ROUTING.md`). Verificado: `src/model-provider/` solo tiene `.gitkeep`, no existe `config/models.yaml`, no hay SDK de LLM instalado en `package.json`. El plan no menciona cómo se generaría el resumen — esto es un prerequisito real, no un detalle menor.
3. **Referencia rota en AGENTS.md §5.1:** cita "ver ADR 0002" para el diseño nonce del sanitizador, pero ADR 0002 es sobre `audit_log`/`request_id` (colisión de numeración heredada del bundle de docs original, antes de que existiera ningún ADR). Cuando se construya el sanitizador, escribir el ADR real (sería el 0004).
4. **Confirmar con el owner** URL de Canvas, token y curso de prueba antes de escribir código (paso explícito de `PROMPTS.md` §3.1, punto 3) — no asumir env vars sin esa conversación.
5. **`canvasScheduleStudyBlock` depende de Google Calendar, que no existe.** No hay módulo de Calendar en el repo. Ya existe un tool stub `createCalendarEvent` (Fase 2.2, sin implementación real) en `src/tools/registry.ts` — aclarar si `canvasScheduleStudyBlock` lo reusa o si son dos tools distintos, y en cualquier caso stubear la dependencia honestamente (patrón `ModalService` 501 de Yormun_Executor) en vez de una implementación parcial silenciosa.
6. **`CANVAS_BASE_URL`/`CANVAS_API_TOKEN` no deberían ser opcionales.** `AGENTS.md` línea 371 exige fail-fast si falta una variable requerida. Marcarlas opcionales permite que el módulo de Canvas quede a medias en silencio — siendo Fase 3 específicamente para habilitar Canvas, deberían ser requeridas.

Lo que el plan sí acierta: los 3 niveles HITL coinciden con blueprint/PROMPTS, el rate limit de 30 req/min es correcto, mockear el API de Canvas en tests es válido (`AGENTS.md` §6.3 permite mocks de APIs externas), y `external_inputs_summary` ya existe en el schema de `audit_log` — no hace falta migración ahí.

## Pull Requests — todos mergeados a `main`

| Repo | PR | Contenido | Estado |
| --- | --- | --- | --- |
| Yormun_Infra | [#1](https://github.com/JFrnck/Yormun_Infra/pull/1) | Fase 1.1: manifests K3s, bootstrap, runbook | ✅ mergeado |
| Yormun_Infra | ~~#2~~ → [#3](https://github.com/JFrnck/Yormun_Infra/pull/3) | Fase 1.2: backups + verify-restore | ✅ mergeado (ver nota abajo) |
| Yormun_Core | [#1](https://github.com/JFrnck/Yormun_Core/pull/1) | Fase 2.1: scaffolding NestJS | ✅ mergeado |
| Yormun_Core | [#2](https://github.com/JFrnck/Yormun_Core/pull/2) | Fase 2.2: HITL classifier + audit log | ✅ mergeado |
| Yormun_Executor | [#1](https://github.com/JFrnck/Yormun_Executor/pull/1) | Fase 2.1: scaffolding NestJS | ✅ mergeado |
| Yormun_Executor | [#2](https://github.com/JFrnck/Yormun_Executor/pull/2) | Fase 2.3: RBAC + ejecución aislada | ✅ mergeado |
| Yormun_Web | [#1](https://github.com/JFrnck/Yormun_Web/pull/1) | Fase 2.1: scaffolding Vite+React | ✅ mergeado |
| Yormun_CLI | [#1](https://github.com/JFrnck/Yormun_CLI/pull/1) | Fase 2.1: scaffolding Ink | ✅ mergeado |
| Yormun_Infra | [#4](https://github.com/JFrnck/Yormun_Infra/pull/4) | RBAC del ServiceAccount de yormun-executor (ADR 0003 punto 3, follow-up de Fase 2.3) | ✅ mergeado |
| Yormun_Core | [#3](https://github.com/JFrnck/Yormun_Core/pull/3) | Prerequisitos Fase 3.1: injection-sanitizer + model-provider (ADR 0004) | ✅ mergeado |
| Yormun_Core | [#4](https://github.com/JFrnck/Yormun_Core/pull/4) | Fase 3.1: integración Canvas LMS + Shadowing Académico | ✅ mergeado |
| Yormun_Core | [#5](https://github.com/JFrnck/Yormun_Core/pull/5) | Fase 2.4: bot de Telegram (grammY + webhook + auth + secret_token) | ✅ mergeado |
| Yormun_Core | [#6](https://github.com/JFrnck/Yormun_Core/pull/6) | Fase 4.1: budget guard + kill switch | ✅ mergeado |
| Yormun_Core | [#7](https://github.com/JFrnck/Yormun_Core/pull/7) | Prerequisito Fase 4.2: 4 tools de Calendar en registry.ts | ✅ mergeado |
| Yormun_Core | [#8](https://github.com/JFrnck/Yormun_Core/pull/8) | Prerequisito Fase 4.2: mecanismo "ejecutar al aprobar" (`ToolExecutorRegistry` + `ApprovalExecutionService`) | ✅ mergeado |
| Yormun_Core | [#9](https://github.com/JFrnck/Yormun_Core/pull/9) | Fase 4.2: integración Google Calendar + Gmail (OAuth Testing, rate limiting, sanitización, diferido HITL) | ✅ mergeado (3 rondas de revisión, ver feedback abajo) |
| Yormun_Core | [#10](https://github.com/JFrnck/Yormun_Core/pull/10) | Fase 4.3: memoria extendida del agente con sqlite-vec | ✅ mergeado |

**Nota — Yormun_Infra #2 se reemplazó por #3:** al mergear #1 con `--delete-branch`, GitHub cerró automáticamente #2 porque su rama base (`feature/claude/infra-base`, la de #1) dejó de existir — efecto colateral no documentado de GitHub en PRs apilados, no una acción intencional. Un PR cerrado así no se puede reabrir ni re-apuntar vía API una vez cerrado. Recuperado abriendo #3 desde la misma rama head (`feature/claude/infra-backups`, intacta) directo contra `main`; contenido idéntico (26 archivos, 1128 inserciones), CI verde, mergeado normalmente.

**Lección para futuros PRs apilados** (aplicada ya en Core y Executor sin incidentes): mergear el PR padre **sin** `--delete-branch` → `gh pr edit <hijo> --base main` → verificar que el hijo quede limpio → **recién entonces** borrar la rama vieja del padre → mergear el hijo con `--delete-branch`.

Los `"name": "temp-*"` de `package.json` en Web y CLI ya no aplican como pendiente — no se volvió a tocar antes del merge; si sigue ahí, es follow-up menor, no bloqueante.

## Decisiones del owner

- **2026-07-19 — ORM: Drizzle** (recomendación de ANALISIS §5 aceptada). En uso en Yormun_Core desde la Fase 2.2 (PR #2).
- **2026-07-19 — Observabilidad Fase 1: mínima** — Prometheus + Loki + Grafana básicos; **Tempo y PgBouncer pospuestos** hasta que duelan.
- **2026-07-21 — Testing: Vitest, no Jest.** Migrado en Core y Executor. **Si Antigravity vuelve a tocar estos repos, NO revertir a Jest.**
- **2026-07-21 — BLUEPRINT 9.5 corregido (ADR 0002):** `audit_log` gana columna `request_id`; estado "pendiente" en tabla mutable separada `pending_approvals`. Implementado en Yormun_Core PR #2.
- **2026-07-21 — ADR 0001 (4 niveles HITL)** escrito e implementado en Yormun_Core PR #2.
- **2026-07-22 — Testing de Executor: K3s real vía testcontainers, no mocks del cliente de Kubernetes** (`@testcontainers/k3s`). El owner eligió explícitamente esta opción por sobre mockear — ver ADR 0003 punto 5. Costo aceptado: CI más lento (~35-75s típico; se observó flakiness de red anidada containerd-en-Docker en algunas corridas, mitigada con reintentos).
- **2026-07-22 — ADR 0003 (Executor: separación + hallazgo `deno eval`)** escrito e implementado en Yormun_Executor PR #2.
- **2026-07-23 — Canvas LMS: single-tenant confirmado, sin soporte multi-usuario.** El owner preguntó si Canvas soportaría "cambiar de usuario" (cuentas de terceros); se le presentaron 3 opciones (single-tenant / multi-cuenta propia / multi-tenant real) y confirmó mantener single-tenant, consistente con BLUEPRINT §1-2 ("plataforma personal", no multi-región/HA). Un solo Personal Access Token de Canvas (del owner) → Infisical → REST API, como ya especifica §7.1. **No implementar** `user_id` en `audit_log`/`pending_approvals`, aislamiento de memory por usuario, ni enrutamiento HITL multi-persona — si en el futuro se reconsidera, requiere un ADR nuevo porque cambia el modelo de seguridad completo del proyecto.
- **2026-07-23 — Prerequisitos de Fase 3.1 (sanitizer + model-provider): los construye Claude Code, no Antigravity.** El plan de Antigravity para Canvas proponía construir `src/security/injection-sanitizer.ts` y `src/model-provider/**` dentro de su propia rama — ambos son área exclusiva de Claude Code por `WORKFLOW.md` §2.2 (infraestructura compartida que futuras integraciones como Gmail/Telegram también necesitarán). El owner confirmó que Claude Code los construyera aparte primero; Antigravity los consume una vez mergeados. Ver PR #3 de Yormun_Core y ADR 0004.
- **2026-07-23 — Después de Fase 3.1, siguiente prioridad: terminar Fase 2.4 (bot de Telegram), no Fase 4.x.** La Tarea A de Fase 2.4 (`model-provider`) ya quedó hecha de rebote en PR #3. Queda solo la Tarea B (`src/telegram/`, grammY). El owner la priorizó sobre Budget guard (4.1)/Google Calendar (4.2)/Memoria (4.3) porque el sistema HITL ya construido no tiene todavía ningún canal real de notificación/aprobación.
- **2026-07-24 — Fase 4.1: alertas de budget/kill switch van directo a Telegram, no vía Alertmanager.** BLUEPRINT 9.6/10.4 especifica "Alertmanager → Telegram", pero Alertmanager nunca se desplegó (Fase 1 observability quedó explícitamente mínima: Prometheus + Loki + Grafana). Desplegarlo ahora era trabajo de infra fuera de alcance de Fase 4.1. Se decidió notificar directo desde `Yormun_Core` al bot ya construido (Fase 2.4) — mismo destino final, sin la dependencia de infra nueva. Documentado en el plan de la fase, no requirió pausar para preguntar.
- **2026-07-24 — Fase 4.1: `sessionId` opcional en `BudgetGuardedModelRouter`, no una abstracción de sesión persistente.** El código no tenía ningún concepto de "sesión" (ningún caller pasaba un ID). Se agregó un `sessionId` opcional — si no se pasa, cada llamada es su propia sesión (UUID nuevo). Telegram usa un `sessionId` fijo por chat (conversación libre comparte presupuesto); Canvas/shadowing no pasa ninguno (cada corrida nocturna es una sola llamada). Evita inventar una abstracción de sesión persistente que nada más en el sistema necesita todavía.
- **2026-07-24 — Fase 4.2: OAuth scopes confirmados por el owner.** `calendar` (lectura/escritura de eventos), `gmail.readonly` (leer correos), `gmail.send` (enviar/responder) — los 3 mínimos necesarios para las tools ya declaradas, sin scopes de administración ni de otras APIs de Workspace.
- **2026-07-24 — Fase 4.2: el gap de "reset de la alerta de OAuth" se corrige antes de mergear PR #9, no se acepta como deuda.** Al revisar el PR se encontró que `updateLastRefreshedAt()` no tenía ningún caller real y la alerta de vencimiento solo logueaba (nunca le llegaba nada al owner). Se le presentaron 2 opciones: mergear igual y dejarlo como follow-up, o pedirle a Antigravity un fix acotado antes de mergear. Eligió lo segundo — mismo criterio que otras veces con hallazgos de seguridad/HITL: si el fix es chico y acotado, se resuelve antes de dar la fase por cerrada en vez de acumular deuda. Resultado: comando `/google-oauth-refreshed` + alerta proactiva real, ver "PR #9 (Google Calendar + Gmail): 3 rondas de revisión".
- **2026-07-24 — Fase 4.2: el hallazgo "no existe mecanismo de ejecutar-al-aprobar" lo construye Claude Code primero, no Antigravity.** Al revisar el plan de Antigravity para Google Calendar/Gmail se detectó que ningún tool `confirm` anterior había tenido efectos reales (Canvas es todo `auto`; `sendEmail` se declaró en Fase 2.2 pero nunca se implementó) — por lo que `TelegramBotService.processApproval()` nunca ejecutaba nada, solo cambiaba estado y auditaba, y `pending_approvals` no guardaba el payload de la acción. Se le presentaron 2 opciones al owner: que Antigravity resolviera esto de forma acotada a sus 2 tools, o que Claude Code construyera el mecanismo genérico primero. Eligió que Claude Code lo construyera primero (mismo criterio que los prerequisitos de Fase 3.1: infraestructura compartida que futuras integraciones también necesitarán). Resultado: PR #8 (`ToolExecutorRegistry` + `ApprovalExecutionService`), ver detalle en la sección "Fase 4.2 — decisiones y prerequisitos".
- **2026-07-25 — Fase 4.3: proveedor de embeddings de la memoria — OpenAI `text-embedding-3-large`, no Voyage AI.** BLUEPRINT §6.4 menciona ambas opciones para el corpus en pgvector (no construido todavía), pero no especifica nada para la memoria sqlite-vec — no había ninguna API key de ninguno de los dos en el proyecto. Se le presentaron 3 opciones al owner: reusar Gemini (`GEMINI_API_KEY` ya configurada, cero vendor nuevo), OpenAI, o Voyage AI. Eligió OpenAI explícitamente pese al costo de integración de un vendor nuevo (`OPENAI_API_KEY` nueva, SDK `openai` nuevo) — no la opción de menor fricción. Truncado a 1024 dimensiones (balance calidad/tamaño para un store ≤100k vectores). Vive en `src/memory/embedding-provider.ts`, no en `src/model-provider/` (esa capacidad es TaskProfile/chat-completion con budget guard; embeddings es aparte, sin riesgo "runaway").

## Plan aprobado

0. **Paso previo** — ✅ hecho.
1. **Fase 1.1** [rol Antigravity, ejecuta Claude Code] — ✅ hecho, PR #1 Yormun_Infra.
2. **Fase 1.2** [Claude Code] — ✅ hecho, PR #2 Yormun_Infra.
3. **Fase 2.1** [rol Antigravity, ejecuta Claude Code] — ✅ hecho, PR #1 en los 4 repos de app.
4. **Fase 2.2** [Claude Code] — ✅ hecho, PR #2 Yormun_Core (HITL classifier + audit log). CI verde.
5. **Fase 2.3** [Claude Code] — ✅ hecho, PR #2 Yormun_Executor (RBAC + ejecución aislada). CI verde, mergeado.
6. **Fase 3.1** [Antigravity] — ✅ hecho, PR #4 Yormun_Core (Canvas LMS + Shadowing Académico). Prerequisitos (`security`/`model-provider`) en PR #3, Claude Code.
7. **Fase 2.4** [Antigravity] — ✅ hecho, PR #5 Yormun_Core (bot de Telegram).
8. **Fase 4.1** [Claude Code] — ✅ hecho, PR #6 Yormun_Core (Budget guard + kill switch).
9. **Fase 4.2** [Antigravity] — ✅ hecho, PR #9 Yormun_Core (Google Calendar + Gmail). Prerequisitos en PR #7 (tools de Calendar en registry.ts) y PR #8 (`ToolExecutorRegistry`/`ApprovalExecutionService`), ambos Claude Code. `canvasScheduleStudyBlock` desbloqueado.
10. **Fase 4.3** [Claude Code] — ✅ hecho, PR #10 Yormun_Core (Memoria extendida del agente con sqlite-vec). Ver sección dedicada abajo.

**Fin de la Fase 2, Fase 3.1, Fase 4.1, Fase 4.2 y Fase 4.3 del roadmap.** Sin fases pendientes asignadas — a la espera de que el owner defina Fase 5+ (`docs/PROMPTS.md` aún no tiene prompts para esas fases).

## Bloqueados / esperando

- Ejecución real del bootstrap en la VM OCI la hace el owner (Claude Code solo escribe manifests/scripts).

## Fase 4.3 — resumen técnico (Yormun_Core PR #10, mergeado 2026-07-25)

`src/memory/` (BLUEPRINT §3.3.1, PROMPTS.md §4.3), área exclusiva de Claude Code:

- **`EmbeddingProvider`**: OpenAI `text-embedding-3-large` truncado a 1024 dims (decisión del owner, ver "Decisiones del owner"). Único consumidor de `OPENAI_API_KEY` (nueva).
- **`MemoryStore`**: `better-sqlite3` + extensión `sqlite-vec` (`vec0`, `distance_metric=cosine`), WAL mode — único lugar del repo que importa ambas libs (restricción explícita de `PROMPTS.md`). KNN brute-force real (`k = COUNT(*)`, sin índice ANN — BLUEPRINT exige no depender de los índices ANN de sqlite-vec, siguen en alpha), filtros de metadata aplicados después del ranking completo. Detalle no obvio encontrado durante la implementación: el `rowid` debe bindearse como `BigInt` (no `number`) al insertar en una tabla `vec0` vía better-sqlite3 — verificado empíricamente, un `number` da `SqliteError: Only integers are allowed for primary key values`.
- **`MemoryService`**: API pública `remember()`/`recall()`/`consolidate()`. Reusa `sanitizeForIndexing()` de `src/security/injection-sanitizer.ts` — esa función ya existía desde Fase 3.1/ADR 0004 explícitamente preparada para este momento ("sin consumidor todavía: `src/memory/**` no existe"), sin necesidad de tocar el sanitizer.
- **`ConsolidationService`**: usa el TaskProfile `memory_consolidation` (ya declarado en `config/models.yaml` desde el bootstrap del repo) + `BudgetGuardedModelRouter`. Parseo del JSON de salida con Zod, falla ruidoso (`ConsolidationParseError`) ante formato inválido en vez de asumir una estructura.
- 97.67% statements / 86.84% branches / 98.76% líneas en `src/memory/` (>85% exigido por `PROMPTS.md`). Incluye el test del criterio de éxito explícito: una preferencia consolidada en una sesión aparece en el `recall()` de la siguiente.

**Handoff pendiente, documentado (no silencioso):** `consolidate()`/`recall()` no están conectados al ciclo de vida real de una sesión de Telegram (`src/telegram/**`) — `PROMPTS.md` §4.3 solo pedía el módulo con estas 3 firmas y tests de round-trip directos. Cuando se priorice, hace falta: llamar `recall()` al inicio de una conversación (inyectar las k entradas más relevantes al contexto) y `consolidate(sessionId, transcript)` al cierre — requiere que `TelegramBotService` importe `MemoryModule` (sin ciclo, `MemoryModule` no depende de Telegram) y decidir qué cuenta como "cierre de sesión" en un chat de Telegram sin un concepto explícito de sesión cerrada hoy. No asignado a nadie todavía.

## PR #9 (Google Calendar + Gmail): 3 rondas de revisión de código, mergeado 2026-07-24

Claude Code revisó el PR #9 de forma independiente en cada ronda (checkout real de la rama + `tsc --noEmit`/`lint`/`test`/`test:integration`/`gh pr checks`, nunca solo el self-report de Antigravity) antes de mergear.

**Ronda 1 (código):**
1. **Bloqueante — CI real en rojo, no verde.** `.github/workflows/ci.yml` nunca se actualizó con las nuevas env vars requeridas (`GOOGLE_CLIENT_ID`/`SECRET`/`REFRESH_TOKEN`) — el self-report de Antigravity se basó en correr los comandos localmente con env vars exportadas a mano, no en el run real de GitHub Actions, que sí fallaba (`pnpm run test:e2e` reventaba al bootstrapear `AppModule`). **Lección repetida** (mismo patrón que ya pasó en Fase 2.4): correr `gh pr checks <N>` siempre antes de reportar "CI verde".
2. **Seguridad — inyección de headers CRLF en `sendEmail`.** `GoogleGmailClientService.sendEmail()` interpolaba `to`/`subject` directo en headers RFC822 sin sanitizar — un `\r\n` embebido podía inyectar headers arbitrarios (`Bcc:`, etc.) invisibles en el `planSummary` que el owner aprueba por Telegram.
3. **Falta test de integración real para `GoogleOAuthService`** — tocaba Postgres de verdad pero solo tenía tests con `Db` mockeado, rompiendo el patrón del resto del repo (`AGENTS.md` §6.3).
4. Menor: `CalendarNotImplementedError` quedó como código muerto tras desbloquear `canvasScheduleStudyBlock`.

Todo resuelto por Antigravity en la siguiente iteración — de paso, al agregar el test de integración real salió a la luz que la migración `0003_google_oauth_state` **nunca se había registrado en `drizzle/meta/_journal.json`**, así que jamás se hubiera aplicado en un deploy real. Corregido también.

**Ronda 2 (cierre del ciclo de alerta OAuth):** el mecanismo de aviso de vencimiento de OAuth no tenía forma de resetearse en producción — `updateLastRefreshedAt()` no tenía ningún caller real, y `checkOAuthTokenStatus()` solo hacía `logger.warn()` sin avisarle al owner por ningún canal real. El owner eligió explícitamente pedir el fix antes de mergear en vez de aceptarlo como deuda. Resuelto: comando `/google-oauth-refreshed` en Telegram (mismo patrón que `/unpause`, Fase 4.1) + alerta proactiva real vía `TelegramBotService.checkOAuthAlert()` (con dedupe diario) cuando pasan ≥6 días sin refresh.

**Deuda menor aceptada, no bloqueante:** el cron viejo `GoogleOAuthService.checkOAuthTokenStatus()` (solo loguea) quedó redundante con `checkOAuthAlert()` de Telegram — limpieza cosmética pendiente, sin urgencia. Falta también un test explícito del dedupe diario de la alerta OAuth (sí existe el equivalente para los umbrales de budget).

## Feedback Ronda 1 (Google Calendar + Gmail) para Antigravity, plan enviado 2026-07-24 (ya resuelto)

Claude Code revisó el plan de Antigravity para `src/integrations/google/` (además de resolver los 2 puntos de diseño de la sección siguiente, que ya se le comunicaron aparte). 3 puntos pendientes de ajustar antes/durante la implementación:

1. **Falta rate limiting.** `PROMPTS.md` §4.2 exige explícitamente "Rate limiting según los límites de Google APIs" como restricción — el plan de `google-calendar-client.service.ts`/`google-gmail-client.service.ts` no menciona ningún limiter, a diferencia de `CanvasClientService` (Fase 3.1) que sí implementa una ventana deslizante de 30 req/min. Aplicar el mismo patrón (ajustado a los límites reales de Calendar/Gmail API, que son más generosos que Canvas pero no infinitos).
2. **Cron de refresh de OAuth con día fijo de la semana no puede modelar "24h antes del vencimiento real".** El plan usa `@Cron('0 9 * * 1')` (todos los lunes). Pero el vencimiento real es 7 días desde el último refresh — que no necesariamente cae en lunes, y se corre en el tiempo cada vez que se hace un refresh real. Un cron de día fijo puede avisar demasiado tarde (o innecesariamente temprano) según cuándo se hizo el último refresh. Correcto: cron diario (`@Cron('0 9 * * *')`) que compara `now` contra un timestamp persistido de "último refresh" + 6 días (aviso a las 24h del vencimiento real), no contra un día de la semana fijo.
3. **`GOOGLE_REFRESH_TOKEN` no debería ser opcional.** Mismo anti-patrón que `TELEGRAM_WEBHOOK_SECRET` en la Ronda 3 de Telegram (ver arriba): "hay que obtenerlo manualmente una vez" no es motivo válido para saltarse fail-fast — ese mismo argumento se usó y se rechazó para otros secrets. Debe ser requerida (`AGENTS.md` línea 371), el bootstrap manual de obtenerlo una vez es un paso de operación, no una razón para permitir que el módulo arranque sin él.

Lo que el plan sí acierta: consumo correcto de `wrapUntrustedContent` para el contenido de emails/eventos, reuso de las tools ya declaradas en `registry.ts` sin tocarlo, y el diseño de OAuth2 con refresh token vía Infisical (no en env plano) es correcto.

## Fase 4.2 — decisiones y prerequisitos (Yormun_Core PR #7)

Antes de que Antigravity implemente `src/integrations/google/`, se resolvieron 2 cosas que `PROMPTS.md` §4.2 no cubre o contradice:

1. **"Borrar evento pasado: notify, borrar evento futuro: confirm" no se puede expresar como una sola tool.** El `hitlLevel` es estático por tool y nunca depende de inputs en runtime (`AGENTS.md` §5.4, probado explícitamente en `classifier.spec.ts`). Se separó en `deleteCalendarEventPast` (notify) y `deleteCalendarEventFuture` (confirm) — el LLM elige cuál invocar según la fecha del evento antes de llamar, no el clasificador después.
2. **`updateCalendarEvent`: nivel HITL no especificado en los docs.** El owner confirmó `notify` (mismo riesgo que crear un evento).

Gmail no suma tools nuevas: "listar/leer" reusa `readEmails` (auto, Fase 2.2), "responder/enviar nuevo" reusa `sendEmail` (confirm, Fase 2.2) — ambos pares ya comparten el nivel HITL correcto, no hay motivo para declarar duplicados. `createCalendarEvent` (Fase 2.2) también se reusa tal cual para "crear evento".

Nota técnica para Antigravity: BLUEPRINT 7.2 menciona "job en BullMQ" para el aviso de refresh de OAuth cada 7 días, pero BullMQ no está instalado en el repo — todos los crons existentes (`chain-verification`, `timeout`, `kill-switch`, `shadowing`) usan `@nestjs/schedule` (`@Cron`). Usar el mismo patrón, no agregar BullMQ para un solo job semanal.

### Prerequisito 3: mecanismo "ejecutar al aprobar" (PR #8, mergeado)

Al revisar el plan de Antigravity se encontró que **no existe ningún mecanismo para ejecutar la acción real de un tool al aprobarse** — `TelegramBotService.processApproval()` (Fase 2.4) solo orquestaba `DualConfirmService.recordApproval()` + `AuditService.recordApproval()`, nunca invocaba lógica de ningún tool. Tampoco se guardaba el payload de la acción (to/subject/body de un email, etc.) en `pending_approvals`. No se había notado antes porque `sendEmail` (único `confirm` declarado hasta ahora) nunca se implementó. Resuelto en PR #8:

- `pendingApprovals` gana columna `payload` (jsonb, nullable).
- `ToolExecutorRegistry` (nuevo, `src/hitl/tool-executor.registry.ts`): registro en memoria `toolName → executor`. Cada módulo de integración se registra en su propio `onModuleInit()` — `src/hitl/` no importa Google/Canvas/etc., evitando ciclos.
- `DualConfirmService.createPendingApproval` acepta `payload?: unknown` opcional.
- `ApprovalExecutionService` (nuevo, `src/hitl/approval-execution.service.ts`): `resolveAndExecute(requestId, approver)` / `resolveRejection(requestId, approver)` — orquesta resolver + ejecutar + auditar + limpiar pendiente en un solo lugar. `TelegramBotService.processApproval/processRejection` ahora delegan acá en vez de orquestar `DualConfirmService`+`AuditService` a mano.

**Handoff a Antigravity:** `sendEmail` y `deleteCalendarEventFuture` (los 2 tools `confirm`/`dual-confirm` reales de esta fase) deben, en el `onModuleInit()` de su propio módulo de Google, llamar `toolExecutorRegistry.register('sendEmail', async (payload) => {...})` con la lógica real de envío/borrado. El handler de la tool, en vez de ejecutar directo, llama `classifyToolCall` y si no es `auto`, crea la pending approval con `dualConfirmService.createPendingApproval({ ..., payload })` (el payload con to/subject/body o eventId) y devuelve "en espera de aprobación" — no ejecuta nada hasta que `ApprovalExecutionService` lo resuelva desde Telegram.

## Fase 2.3 (follow-up) — RBAC de NetworkPolicy en Yormun_Infra (PR #4)

Cierra ADR 0003 punto 3. `k8s/base/executor/`: `ServiceAccount` `executor` en `yormun-executor`, `Role` `executor-agents-sandbox` en `agents-sandbox` (`create/get/list/delete` de Pods per BLUEPRINT 4.2 + `create/delete` de NetworkPolicies, la extensión que ADR 0003 identificó como necesaria), `RoleBinding` conectando ambos across namespaces. Sin acceso a Secrets/ConfigMaps/Deployments/otros namespaces. Verificado con `kubectl kustomize k8s/base` + CI verde (lint manifests, shellcheck, bats, GitGuardian).

## Fase 2.3 — resumen técnico (Yormun_Executor PR #2)

Implementado según ADR 0003 (separación de privilegios + hallazgo de seguridad `deno eval`):

- `src/rbac/` — whitelist propia (`runCode`, sin egreso), `RbacValidatorService` como única puerta de entrada (403 si no está en whitelist).
- `src/k8s/k8s.service.ts` — único punto de contacto con `@kubernetes/client-node` en todo el proyecto; `namespace` fijo al construir, estructuralmente no puede tocar otro namespace.
- **Hallazgo de seguridad real:** verificado contra Deno 2.9.3 real que `deno eval` tiene *"implicit access to all permissions"* — ignora `--allow-net` completamente. Corregido a `deno run` + código como `data:` URL en base64 (sin ConfigMap, sin shell).
- `src/pod-lifecycle/` — ciclo de vida bajo demanda, sin warm pool, siempre destruye el pod.
- `src/modal/` — stub explícito (501) para `remote: true`, Fase 5.
- **Tests:** 30 unitarios + 5 e2e (capa HTTP completa sin K8s real) + 2 de integración con **K3s real** (`@testcontainers/k3s`): camino feliz de ejecución, y el test de aislamiento de red real que pide PROMPTS.md (un pod en `agents-sandbox` con permiso de red de Deno explícito igual no alcanza un servicio en `yormun`, bloqueado por la NetworkPolicy real).
- **Fix incidental:** `app.controller.ts` (heredado de Fase 2.1) tenía un endpoint stub `@Post('execute')` que colisionaba de ruta con el `ExecuteController` real — toda petición a `/execute` devolvía 201 del stub, saltándose RBAC/Zod por completo. Corregido.

## Fase 2.2 — resumen técnico (Yormun_Core PR #2)

Implementado según ADR 0001 (4 niveles HITL) y ADR 0002 (`request_id` + `pending_approvals`):

- `src/tools/registry.ts` — 3 tools stub con hitlLevel estático, `Object.freeze()`-ado.
- `src/hitl/` — `classifier.ts` (nunca decide por `inputs`, tool desconocida → `UnknownToolError`), `dual-confirm.service.ts` (estado persistido en Postgres, 30s reales entre aprobaciones), `timeout.service.ts` (nunca aprueba; descarta o escala/abandona).
- `src/audit/` — `hash-chain.ts` (puro), `audit.service.ts` (insert-only, advisory lock para serializar escrituras concurrentes entre réplicas durante rolling update), `chain-verification.service.ts` (cron diario, bloquea escrituras si detecta corrupción).
- **Infra nueva que no existía:** Drizzle + `pg` (primera vez que Core habla con Postgres), migraciones con `.down.sql` a mano, `src/config/` (Zod + `@nestjs/config`, fail-fast).
- **Tests:** 36 unitarios (100% matriz tool×nivel, 4 tests de mutación del hash chain) + 18 de integración con testcontainers (Postgres real) + 1 e2e.
- **Gap de cobertura documentado:** `timeout.service.ts` en ~70% — los caminos 'escalate'/'abandon' no tienen integration test porque ninguna tool registrada usa `timeoutBehavior: 'escalate'` todavía (llega con Canvas, Fase 3).

## CI de Yormun_Core / Yormun_Executor: bugs encontrados y corregidos (2026-07-21)

El owner reportó que el CI de ambos repos falló al pushear la Fase 2.1. Investigación encontró **cuatro problemas reales**, todos corregidos:

1. **Causa raíz del fallo de CI:** pnpm 11 bloquea por defecto scripts de instalación no aprobados (`ERR_PNPM_IGNORED_BUILDS`). Fix: `pnpm-workspace.yaml` con `allowBuilds`.
2. **Grave, independiente del CI:** `node_modules/` (289M) y `dist/` estaban commiteados en git en ambos repos. Corregido: `.gitignore` estándar + `git rm --cached`.
3. **Bug funcional real en Executor:** `app.controller.ts` importaba `AppService` desde el archivo equivocado y tipaba la dependencia como `any` — rompía el DI de Nest en runtime.
4. **`@typescript-eslint/no-explicit-any` estaba en `'off'`** en el scaffolding — contradice AGENTS.md 3.2. Reactivado en `'error'`.

También corregido de paso: glob patterns rotos en `lint`/`format`, y `package.json` `name` de ambos repos.

**CI verificado en verde** en Core PR #1/#2 y Executor PR #1; Executor PR #2 esperando confirmación final.

## Recientemente completado (últimos 7 días)

- 2026-07-24: [Yormun_Core] [PR #7](https://github.com/JFrnck/Yormun_Core/pull/7) mergeado — prerequisito de Fase 4.2: 4 tools de Calendar declaradas en `registry.ts` (`listCalendarEvents` auto, `updateCalendarEvent` notify, `deleteCalendarEventPast` notify, `deleteCalendarEventFuture` confirm). Matriz HITL actualizada a 10 tools, 100% cobertura.
- 2026-07-24: [Yormun_Core] [PR #6](https://github.com/JFrnck/Yormun_Core/pull/6) mergeado — Fase 4.1 completa: `src/budget/` (tracking sesión/día, degradación al 80% vía el hint `budgetRemaining` de Fase 3.1, kill switch de runaway persistido en Postgres), métricas Prometheus, `BudgetGuardedModelRouter` ya inyectado en Canvas y Telegram, `/budget` con datos reales y `/unpause` nuevo en el bot. 87 tests nuevos (unitarios + integración Postgres real), CI verde, verificado con `tsc --noEmit -p tsconfig.json` de forma independiente antes de mergear.
- 2026-07-24: [Yormun_Core] [PR #5](https://github.com/JFrnck/Yormun_Core/pull/5) mergeado — Fase 2.4 completa: bot de Telegram con grammY en modo webhook, auth estricta con `TELEGRAM_OWNER_CHAT_ID` numérico, `TELEGRAM_WEBHOOK_SECRET` requerida fail-fast validando el header de Telegram, `bot.init()` real, comandos `/start`/`/status`/`/tasks`/`/approve`/`/reject`/`/budget` (stub honesto), integración con `DualConfirmService`/`AuditService`/`ModelRouterService`. 3 rondas de review contra el código real (bug de tipos chat_id, gap de seguridad del webhook, `botInfo` hardcodeado, tipos en specs invisibles a `pnpm build`), todas resueltas y verificadas independientemente antes de mergear. 105 tests en verde.
- 2026-07-23: [Yormun_Core] [PR #4](https://github.com/JFrnck/Yormun_Core/pull/4) mergeado — Fase 3.1 completada por Antigravity (integración Canvas LMS, cliente REST con rate limit de 30 req/min, handlers sanitizados con `wrapUntrustedContent`, ShadowingService nocturno consumiendo `ModelRouterService.complete('long_context', ...)` con Gemini 3.1 Pro, `CalendarNotImplementedError` 501). 95 tests en verde. Claude Code verificó de forma independiente (lint/test/build/typecheck a mano, no solo el self-report) antes de mergear — sin hallazgos nuevos.
- 2026-07-23: [Yormun_Core] PR #3 mergeado — prerequisitos de Fase 3.1: `src/security/injection-sanitizer.ts` (ADR 0004) + `src/model-provider/**` (router, failover, providers Anthropic/Google, config/models.yaml) + declaración de las 3 tools de Canvas en `registry.ts`. 87 tests, cobertura 97% en los módulos nuevos. Fix incidental de CI (env vars faltantes). Antigravity desbloqueado para retomar `integrations/canvas/**`.
- 2026-07-23: [Yormun_Infra] PR #4 mergeado — RBAC de NetworkPolicy para el ServiceAccount del Executor (ADR 0003 punto 3), cerrando el último follow-up técnico de la Fase 2.
- 2026-07-23: Review y merge de los 8 PRs de las Fases 1-2 en los 6 repos. Incidente: Yormun_Infra #2 auto-cerrado por GitHub al borrar la rama base de un PR apilado; recuperado como #3. Procedimiento corregido aplicado sin incidentes en Core y Executor. Todos los repos en `main` limpio, 0 PRs abiertos.
- 2026-07-22: [Yormun_Executor] Fase 2.3 completa (RBAC + ejecución aislada con K3s real). PR #2 abierto. 30 unitarios + 5 e2e + 2 de integración (K3s real vía testcontainers).
- 2026-07-22: [Yormun_Docs] ADR 0003 (Executor: separación + hallazgo `deno eval`) escrito.
- 2026-07-21: [Yormun_Core] Fase 2.2 completa (HITL classifier + audit log + Drizzle + config module). PR #2 abierto, CI verde.
- 2026-07-21: [Yormun_Docs] ADR 0001 (4 niveles HITL) y ADR 0002 (`request_id`/`pending_approvals`) escritos; BLUEPRINT 9.5 corregido.
- 2026-07-21: [Yormun_Core, Yormun_Executor] Migración Jest→Vitest + fix de CI + fix de node_modules/dist trackeados + fix de bug de DI en Executor + reactivación de `no-explicit-any` + fix de globs de lint/format + rename de `package.json`.
- 2026-07-21: PRs abiertos en los 4 repos de app para la Fase 2.1.
- 2026-07-21: [Yormun_Infra] PR #2 abierto (Fase 1.2, backups).
- 2026-07-20: [Yormun_Infra] Fase 1.1 (infra base) terminada; PR #1 abierto.
- 2026-07-20: [Yormun_Core, Yormun_Executor, Yormun_Web, Yormun_CLI] Fase 2.1 completada por Antigravity (scaffolding base de apps y CI pipelines), commiteada en local.
- 2026-07-19: [Yormun_Docs] Bundle de documentación inicial commiteado (`main`).
- 2026-07-19: [todos los repos] Repos creados con README inicial; stubs AGENTS.md/CLAUDE.md colocados.
