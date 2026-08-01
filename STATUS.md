# STATUS

## Última actualización: 2026-08-01 (America/Lima) — actualización 12

> **Antigravity ya está activo** (ver abajo, Fase 3.1/2.4/4.2). Retoma ownership normal de `docs/WORKFLOW.md` sección 2 — Claude Code ya no asume tareas `[ANTIGRAVITY]` por defecto, salvo negociación puntual vía esta misma nota.

## En progreso

### Claude Code

- **Repo:** Jin_Core
- **Rama:** `feature/claude/agent-orchestration` — [PR #15](https://github.com/JFrnck/Jin_Core/pull/15)
- **Descripción:** Fase 5.4 — Orquestación multi-agente: `OrchestratorService` descompone un objetivo en tickets del task ledger (Postgres) y los delega a sub-agentes (instancias adicionales de `AgentService.runTurn()`, Fase 5.1) que corren en paralelo por lotes de dependencias. Ver detalle técnico abajo y ADR 0005.
- **Estado:** 🟢 PR #15 abierto, mergeable, CI verde (`build` & `GitGuardian` ambos en `pass`, verificado con `gh pr checks` real). 284/284 unitarios + 59/59 integración (Postgres real) pasados.
- **Detalle de implementación:**
  - 3 tablas nuevas (`agent_orchestration_runs`, `agent_tickets`, `agent_ticket_comments`) — migración `0004_agent_ledger` (numeración real de `main`; ver nota de coordinación abajo).
  - Descomposición y reconciliación vía LLM (`reasoning_heavy`, Opus 4.8 por defecto), mismo patrón LLM+Zod que `ConsolidationService` (Fase 4.3).
  - Conflictos de bajo riesgo entre sub-agentes se resuelven en modo-auto (comentario de resolución registrado); conflictos materiales escalan al owner reusando el HITL/Telegram existente vía tool sintética `resolveAgentConflict` (`confirm`) — sin canal de escalamiento nuevo.
  - `mergeAgentBranch` declarada como guardrail (PROMPTS.md §5.4) con executor 501 documentado (`AgentBranchMergeNotImplementedError`): hoy ninguna tool le da a un sub-agente la capacidad de producir una branch de código real. Handoff explícito, no silencioso.
  - Kill switch a mitad de un run corta TODO (chequeo antes de cada lote + `KillSwitchActiveError` real propagada).
  - Cambios mínimos/retrocompatibles a Fase 5.1: `AgentTurnInput` gana `allowedTools?`/`actorLabel?` opcionales; `BudgetService` gana `getSessionUsage()` público.
  - Bug real encontrado y corregido durante el smoke test de `AppModule` completo (env vars fake, no solo `tsc`/`test`/`test:integration`): `AGENT_CONFIG` no estaba exportado desde `AgentModule`, `OrchestratorModule` no podía resolverlo — Nest lo hubiera reventado recién al bootstrapear en real. Corregido exportando el token.
  - ADR 0005 escrito (`docs/adr/0005-multi-agent-orchestration.md`).
- **Nota de coordinación — numeración de migración:** esta rama parte de `main` (sin el PR #14 de Antigravity, Fase 5.3, todavía sin mergear), así que la migración nueva es `0004_agent_ledger` — el mismo número que `0004_telegram_sessions` de PR #14. Cualquiera de los dos PRs que se mergee segundo debe renombrar su migración a `0005_*` y actualizar `drizzle/meta/_journal.json` antes o al mergear (mismo tipo de coordinación que ya documentó el incidente de Jin_Infra #2/#3).
- **Próximo:** ninguno activo — Fase 5.5 (pods de servicio) queda libre para arrancar; Fase 6.1 (auth + API) desbloqueada tras 5.4.

### Antigravity

- **Repo:** Jin_Core
- **Rama:** `feature/antigravity/telegram-sessions` — [PR #14](https://github.com/JFrnck/Jin_Core/pull/14) (Commits `ecbe045a` & `1775583f`)
- **Descripción:** Fase 5.3 — Sesiones reales en Telegram (agent loop + memoria extendida + registro de 8 executors a nivel client + reconstrucción de historial multi-turno y persistencia DB de sesión).
- **Estado:** 🟢 PR #14 abierto y CI completamente verde (`build` & `GitGuardian` ambos en `pass`). 265/265 pruebas unitarias pasadas.
- **Detalle de implementación:**
  - Registro de los 8 ejecutores de herramientas faltantes a nivel `ClientService` en `ToolExecutorRegistry` (Google Gmail/Calendar y Canvas) para evitar doble log de auditoría y doble sanitizado.
  - Tabla Postgres `telegram_sessions` + migración `0004_telegram_sessions` registrada en `_journal.json`.
  - Integración del Agent Loop (`AgentService.runTurn()`) y Memoria Extendida (`MemoryService`) en Telegram. Reconstrucción incremental del transcript por turno en Postgres.
  - Ventana deslizante (últimos 20 mensajes + memorias sintéticas si se recuperan) para la llamada al modelo, pero consolidación con el **transcript COMPLETO** en Postgres al cerrar la sesión (`/endsession` o cron `@Cron('*/5 * * * *') checkSessionInactivity()`).
  - Comando `/memory <query>` para consultar memorias a largo plazo en Telegram.
  - Restauradas las pruebas unitarias de idempotencia de `/unpause` y propagación de `KillSwitchActiveError` en el Agent Loop (**28 pruebas pasadas en `telegram-bot.service.spec.ts`, 265 pasadas en total**).








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
2. **El stub de `canvasScheduleStudyBlock` usa la excepción equivocada.** El plan dice seguir "el patrón `ModalService`", pero ese patrón (`Jin_Executor/src/modal/errors.ts`) es `ModalNotImplementedError extends JinError` — clase propia con `code`/`httpStatus`/`cause` — no `NotImplementedException` de `@nestjs/common`. `AGENTS.md` §8.1 exige que todo error tenga su clase custom extendiendo `JinError`. Debe ser `CalendarNotImplementedError extends JinError` (`code: 'CANVAS_CALENDAR_NOT_IMPLEMENTED'`, `httpStatus: 501`) en `src/integrations/canvas/errors.ts`.
3. **Falta sanitizar `canvasListAssignments`, no solo `canvasGetCourseContent`.** El plan envuelve con `wrapUntrustedContent` el resultado de `canvasGetCourseContent` pero no el de `canvasListAssignments` — y esa tool también es `hitlLevel: 'auto'` (su resultado vuelve directo al contexto del LLM como resultado de tool-call). Títulos/descripciones de tareas son contenido externo igual que el contenido de curso — `AGENTS.md` §5.1 exige envolver ambas.

Menor (no bloqueante): usar `.spec.ts` co-ubicado para los tests, no una carpeta `__tests__/` separada — es la convención del resto del repo (hitl, audit, model-provider).

Lo que el plan sí acierta en esta ronda: consumo correcto del sanitizer y el model-router sin reimplementarlos, reuso correcto de las 3 tools ya declaradas en `registry.ts`, `CANVAS_BASE_URL`/`CANVAS_API_TOKEN` requeridas con fail-fast, rate limit de 30 req/min, mocks HTTP en tests, y la referencia a "Fase 4.2" (`PROMPTS.md`) para Google Calendar es correcta.

## Feedback Ronda 1 para Antigravity — plan Fase 3.1 (Canvas), enviado 2026-07-23 (ya resuelto)

Claude Code revisó el plan propuesto (Canvas client + tools + shadowing cron) contra `AGENTS.md`, `docs/PROMPTS.md` §3.1, las golden rules de `docs/BLUEPRINT.md` §15 y el estado real del código en `Jin_Core`. Ajustar antes de implementar:

1. **Sanitizador de injection incorrecto.** El plan propone un `canvas-sanitizer.ts` propio con tag genérico `<untrusted_content>`. `AGENTS.md` §5.1 (golden rule #6) exige una función **compartida** `wrapUntrustedContent(content, source, sessionNonce)` en `src/security/injection-sanitizer.ts`, con tag `<untrusted_content_{sessionNonce}>` (nonce por sesión + escape HTML) y `generateSessionNonce()`. Ese módulo **no existe todavía** (`src/security/` solo tiene `.gitkeep`). Además `Jin_Docs/CLAUDE.md` §3 asigna `src/security/**` a Claude Code, no a Antigravity — coordinar antes de tocarlo, no reimplementar una versión propia y más débil dentro de `integrations/canvas/`.
2. **Falta la infraestructura de model routing.** `PROMPTS.md` §3.1 exige usar Gemini 3.1 Pro para el resumen del shadowing; golden rule #5 prohíbe modelos hardcoded — todo debe pasar por `model-provider/router.ts` + `config/models.yaml` (ver `docs/MODEL_ROUTING.md`). Verificado: `src/model-provider/` solo tiene `.gitkeep`, no existe `config/models.yaml`, no hay SDK de LLM instalado en `package.json`. El plan no menciona cómo se generaría el resumen — esto es un prerequisito real, no un detalle menor.
3. **Referencia rota en AGENTS.md §5.1:** cita "ver ADR 0002" para el diseño nonce del sanitizador, pero ADR 0002 es sobre `audit_log`/`request_id` (colisión de numeración heredada del bundle de docs original, antes de que existiera ningún ADR). Cuando se construya el sanitizador, escribir el ADR real (sería el 0004).
4. **Confirmar con el owner** URL de Canvas, token y curso de prueba antes de escribir código (paso explícito de `PROMPTS.md` §3.1, punto 3) — no asumir env vars sin esa conversación.
5. **`canvasScheduleStudyBlock` depende de Google Calendar, que no existe.** No hay módulo de Calendar en el repo. Ya existe un tool stub `createCalendarEvent` (Fase 2.2, sin implementación real) en `src/tools/registry.ts` — aclarar si `canvasScheduleStudyBlock` lo reusa o si son dos tools distintos, y en cualquier caso stubear la dependencia honestamente (patrón `ModalService` 501 de Jin_Executor) en vez de una implementación parcial silenciosa.
6. **`CANVAS_BASE_URL`/`CANVAS_API_TOKEN` no deberían ser opcionales.** `AGENTS.md` línea 371 exige fail-fast si falta una variable requerida. Marcarlas opcionales permite que el módulo de Canvas quede a medias en silencio — siendo Fase 3 específicamente para habilitar Canvas, deberían ser requeridas.

Lo que el plan sí acierta: los 3 niveles HITL coinciden con blueprint/PROMPTS, el rate limit de 30 req/min es correcto, mockear el API de Canvas en tests es válido (`AGENTS.md` §6.3 permite mocks de APIs externas), y `external_inputs_summary` ya existe en el schema de `audit_log` — no hace falta migración ahí.

## Pull Requests — todos mergeados a `main`

| Repo | PR | Contenido | Estado |
| --- | --- | --- | --- |
| Jin_Infra | [#1](https://github.com/JFrnck/Jin_Infra/pull/1) | Fase 1.1: manifests K3s, bootstrap, runbook | ✅ mergeado |
| Jin_Infra | ~~#2~~ → [#3](https://github.com/JFrnck/Jin_Infra/pull/3) | Fase 1.2: backups + verify-restore | ✅ mergeado (ver nota abajo) |
| Jin_Core | [#1](https://github.com/JFrnck/Jin_Core/pull/1) | Fase 2.1: scaffolding NestJS | ✅ mergeado |
| Jin_Core | [#2](https://github.com/JFrnck/Jin_Core/pull/2) | Fase 2.2: HITL classifier + audit log | ✅ mergeado |
| Jin_Executor | [#1](https://github.com/JFrnck/Jin_Executor/pull/1) | Fase 2.1: scaffolding NestJS | ✅ mergeado |
| Jin_Executor | [#2](https://github.com/JFrnck/Jin_Executor/pull/2) | Fase 2.3: RBAC + ejecución aislada | ✅ mergeado |
| Jin_Web | [#1](https://github.com/JFrnck/Jin_Web/pull/1) | Fase 2.1: scaffolding Vite+React | ✅ mergeado |
| Jin_CLI | [#1](https://github.com/JFrnck/Jin_CLI/pull/1) | Fase 2.1: scaffolding Ink | ✅ mergeado |
| Jin_Infra | [#4](https://github.com/JFrnck/Jin_Infra/pull/4) | RBAC del ServiceAccount de jin-executor (ADR 0003 punto 3, follow-up de Fase 2.3) | ✅ mergeado |
| Jin_Core | [#3](https://github.com/JFrnck/Jin_Core/pull/3) | Prerequisitos Fase 3.1: injection-sanitizer + model-provider (ADR 0004) | ✅ mergeado |
| Jin_Core | [#4](https://github.com/JFrnck/Jin_Core/pull/4) | Fase 3.1: integración Canvas LMS + Shadowing Académico | ✅ mergeado |
| Jin_Core | [#5](https://github.com/JFrnck/Jin_Core/pull/5) | Fase 2.4: bot de Telegram (grammY + webhook + auth + secret_token) | ✅ mergeado |
| Jin_Core | [#6](https://github.com/JFrnck/Jin_Core/pull/6) | Fase 4.1: budget guard + kill switch | ✅ mergeado |
| Jin_Core | [#7](https://github.com/JFrnck/Jin_Core/pull/7) | Prerequisito Fase 4.2: 4 tools de Calendar en registry.ts | ✅ mergeado |
| Jin_Core | [#8](https://github.com/JFrnck/Jin_Core/pull/8) | Prerequisito Fase 4.2: mecanismo "ejecutar al aprobar" (`ToolExecutorRegistry` + `ApprovalExecutionService`) | ✅ mergeado |
| Jin_Core | [#9](https://github.com/JFrnck/Jin_Core/pull/9) | Fase 4.2: integración Google Calendar + Gmail (OAuth Testing, rate limiting, sanitización, diferido HITL) | ✅ mergeado (3 rondas de revisión, ver feedback abajo) |
| Jin_Core | [#10](https://github.com/JFrnck/Jin_Core/pull/10) | Fase 4.3: memoria extendida del agente con sqlite-vec | ✅ mergeado |
| Jin_Core | [#11](https://github.com/JFrnck/Jin_Core/pull/11) | Fase 5.1: agent loop con tool-calling, plan-and-solve y self-correction | ✅ mergeado |
| Jin_Executor | [#3](https://github.com/JFrnck/Jin_Executor/pull/3) | Fase 5.2: `ModalService` real (sandbox Python/pandas) + ruteo local/remoto por `language` | ✅ mergeado |
| Jin_Core | [#12](https://github.com/JFrnck/Jin_Core/pull/12) | Fase 5.2: tool `runCode` en registry + `src/executor-client/` (cierra el loop con el Executor) | ✅ mergeado |

**Nota — Jin_Infra #2 se reemplazó por #3:** al mergear #1 con `--delete-branch`, GitHub cerró automáticamente #2 porque su rama base (`feature/claude/infra-base`, la de #1) dejó de existir — efecto colateral no documentado de GitHub en PRs apilados, no una acción intencional. Un PR cerrado así no se puede reabrir ni re-apuntar vía API una vez cerrado. Recuperado abriendo #3 desde la misma rama head (`feature/claude/infra-backups`, intacta) directo contra `main`; contenido idéntico (26 archivos, 1128 inserciones), CI verde, mergeado normalmente.

**Lección para futuros PRs apilados** (aplicada ya en Core y Executor sin incidentes): mergear el PR padre **sin** `--delete-branch` → `gh pr edit <hijo> --base main` → verificar que el hijo quede limpio → **recién entonces** borrar la rama vieja del padre → mergear el hijo con `--delete-branch`.

Los `"name": "temp-*"` de `package.json` en Web y CLI ya no aplican como pendiente — no se volvió a tocar antes del merge; si sigue ahí, es follow-up menor, no bloqueante.

## Decisiones del owner

- **2026-07-19 — ORM: Drizzle** (recomendación de ANALISIS §5 aceptada). En uso en Jin_Core desde la Fase 2.2 (PR #2).
- **2026-07-19 — Observabilidad Fase 1: mínima** — Prometheus + Loki + Grafana básicos; **Tempo y PgBouncer pospuestos** hasta que duelan.
- **2026-07-21 — Testing: Vitest, no Jest.** Migrado en Core y Executor. **Si Antigravity vuelve a tocar estos repos, NO revertir a Jest.**
- **2026-07-21 — BLUEPRINT 9.5 corregido (ADR 0002):** `audit_log` gana columna `request_id`; estado "pendiente" en tabla mutable separada `pending_approvals`. Implementado en Jin_Core PR #2.
- **2026-07-21 — ADR 0001 (4 niveles HITL)** escrito e implementado en Jin_Core PR #2.
- **2026-07-22 — Testing de Executor: K3s real vía testcontainers, no mocks del cliente de Kubernetes** (`@testcontainers/k3s`). El owner eligió explícitamente esta opción por sobre mockear — ver ADR 0003 punto 5. Costo aceptado: CI más lento (~35-75s típico; se observó flakiness de red anidada containerd-en-Docker en algunas corridas, mitigada con reintentos).
- **2026-07-22 — ADR 0003 (Executor: separación + hallazgo `deno eval`)** escrito e implementado en Jin_Executor PR #2.
- **2026-07-23 — Canvas LMS: single-tenant confirmado, sin soporte multi-usuario.** El owner preguntó si Canvas soportaría "cambiar de usuario" (cuentas de terceros); se le presentaron 3 opciones (single-tenant / multi-cuenta propia / multi-tenant real) y confirmó mantener single-tenant, consistente con BLUEPRINT §1-2 ("plataforma personal", no multi-región/HA). Un solo Personal Access Token de Canvas (del owner) → Infisical → REST API, como ya especifica §7.1. **No implementar** `user_id` en `audit_log`/`pending_approvals`, aislamiento de memory por usuario, ni enrutamiento HITL multi-persona — si en el futuro se reconsidera, requiere un ADR nuevo porque cambia el modelo de seguridad completo del proyecto.
- **2026-07-23 — Prerequisitos de Fase 3.1 (sanitizer + model-provider): los construye Claude Code, no Antigravity.** El plan de Antigravity para Canvas proponía construir `src/security/injection-sanitizer.ts` y `src/model-provider/**` dentro de su propia rama — ambos son área exclusiva de Claude Code por `WORKFLOW.md` §2.2 (infraestructura compartida que futuras integraciones como Gmail/Telegram también necesitarán). El owner confirmó que Claude Code los construyera aparte primero; Antigravity los consume una vez mergeados. Ver PR #3 de Jin_Core y ADR 0004.
- **2026-07-23 — Después de Fase 3.1, siguiente prioridad: terminar Fase 2.4 (bot de Telegram), no Fase 4.x.** La Tarea A de Fase 2.4 (`model-provider`) ya quedó hecha de rebote en PR #3. Queda solo la Tarea B (`src/telegram/`, grammY). El owner la priorizó sobre Budget guard (4.1)/Google Calendar (4.2)/Memoria (4.3) porque el sistema HITL ya construido no tiene todavía ningún canal real de notificación/aprobación.
- **2026-07-24 — Fase 4.1: alertas de budget/kill switch van directo a Telegram, no vía Alertmanager.** BLUEPRINT 9.6/10.4 especifica "Alertmanager → Telegram", pero Alertmanager nunca se desplegó (Fase 1 observability quedó explícitamente mínima: Prometheus + Loki + Grafana). Desplegarlo ahora era trabajo de infra fuera de alcance de Fase 4.1. Se decidió notificar directo desde `Jin_Core` al bot ya construido (Fase 2.4) — mismo destino final, sin la dependencia de infra nueva. Documentado en el plan de la fase, no requirió pausar para preguntar.
- **2026-07-24 — Fase 4.1: `sessionId` opcional en `BudgetGuardedModelRouter`, no una abstracción de sesión persistente.** El código no tenía ningún concepto de "sesión" (ningún caller pasaba un ID). Se agregó un `sessionId` opcional — si no se pasa, cada llamada es su propia sesión (UUID nuevo). Telegram usa un `sessionId` fijo por chat (conversación libre comparte presupuesto); Canvas/shadowing no pasa ninguno (cada corrida nocturna es una sola llamada). Evita inventar una abstracción de sesión persistente que nada más en el sistema necesita todavía.
- **2026-07-24 — Fase 4.2: OAuth scopes confirmados por el owner.** `calendar` (lectura/escritura de eventos), `gmail.readonly` (leer correos), `gmail.send` (enviar/responder) — los 3 mínimos necesarios para las tools ya declaradas, sin scopes de administración ni de otras APIs de Workspace.
- **2026-07-24 — Fase 4.2: el gap de "reset de la alerta de OAuth" se corrige antes de mergear PR #9, no se acepta como deuda.** Al revisar el PR se encontró que `updateLastRefreshedAt()` no tenía ningún caller real y la alerta de vencimiento solo logueaba (nunca le llegaba nada al owner). Se le presentaron 2 opciones: mergear igual y dejarlo como follow-up, o pedirle a Antigravity un fix acotado antes de mergear. Eligió lo segundo — mismo criterio que otras veces con hallazgos de seguridad/HITL: si el fix es chico y acotado, se resuelve antes de dar la fase por cerrada en vez de acumular deuda. Resultado: comando `/google-oauth-refreshed` + alerta proactiva real, ver "PR #9 (Google Calendar + Gmail): 3 rondas de revisión".
- **2026-07-24 — Fase 4.2: el hallazgo "no existe mecanismo de ejecutar-al-aprobar" lo construye Claude Code primero, no Antigravity.** Al revisar el plan de Antigravity para Google Calendar/Gmail se detectó que ningún tool `confirm` anterior había tenido efectos reales (Canvas es todo `auto`; `sendEmail` se declaró en Fase 2.2 pero nunca se implementó) — por lo que `TelegramBotService.processApproval()` nunca ejecutaba nada, solo cambiaba estado y auditaba, y `pending_approvals` no guardaba el payload de la acción. Se le presentaron 2 opciones al owner: que Antigravity resolviera esto de forma acotada a sus 2 tools, o que Claude Code construyera el mecanismo genérico primero. Eligió que Claude Code lo construyera primero (mismo criterio que los prerequisitos de Fase 3.1: infraestructura compartida que futuras integraciones también necesitarán). Resultado: PR #8 (`ToolExecutorRegistry` + `ApprovalExecutionService`), ver detalle en la sección "Fase 4.2 — decisiones y prerequisitos".
- **2026-07-25 — Ejecución autónoma orientada a objetivos + multi-agente: requisito nuevo del owner, incorporado al roadmap.** El owner pidió explícitamente (a) que el agente itere sobre sí mismo con plan-and-solve y self-correction (incorporado como requisito de Fase 5.1, no fase aparte — es la naturaleza del mismo loop), y (b) que el orquestador pueda delegar a dos o más sub-agentes que se coordinen sin discrepar (Fase 5.4 nueva). Decisión de diseño fijada: los sub-agentes NO se comunican peer-to-peer — se coordinan vía el orquestador con un task ledger compartido (hub-and-spoke/blackboard), porque la comunicación libre A2A produce exactamente las discrepancias que se quieren evitar, multiplica el riesgo runaway y rompe el embudo único de HITL/audit. Profundidad de delegación = 1 (un sub-agente no crea sub-agentes). Requiere ADR nuevo al implementarse.
- **2026-07-25 — Refinamiento del ledger (owner): tablero estilo Jira persistido + escalera de decisión + branches seguras.** El owner refinó el diseño de 5.4: el ledger es un tablero tipo Jira (tickets con hilo de comentarios donde los agentes sí se responden entre sí — a través del tablero, nunca por canal privado), **persistido en Postgres** (un objetivo puede quedar días bloqueado en un HITL y debe sobrevivir restarts; el dashboard de Fase 6 lo renderiza como board). Las discrepancias entre agentes se registran como conflicto visible, no se suprimen, y las resuelve una escalera explícita: bajo riesgo → el orquestador decide en modo-auto dejando registrado el porqué; conflicto material o de nivel confirm → escala al owner vía HITL (el owner es siempre el nivel máximo). Trabajo sobre código: cada sub-agente en su propia branch (`feature/agent/<ticket>`), jamás en main; el merge a main es una tool `confirm` aprobada por el owner.
- **2026-07-25 — Requisito nuevo del owner: los agentes pueden levantar apps corriendo (dev servers, backends) con puertos expuestos — Fase 5.5 nueva.** Hoy el Executor solo soporta pods run-to-completion; no existe forma de correr `npm run dev` ni servicios de larga vida. Se agregó la Fase 5.5 [Claude Code]: pods de servicio en `agents-sandbox` + Service + IngressRoute bajo `https://<slug>.jinserver.com` (la regla de oro #10 ya lo anticipaba: una preview app corriendo ES contenido generado por agentes, jamás en jeanfranck.com), con TTL obligatorio + reaper, sin secretos, egress whitelist, `startPreviewService` con hitlLevel `confirm`. Amplía el RBAC del Executor (Services/IngressRoutes solo en agents-sandbox) — requiere ADR.
- **2026-07-31 — El producto pasa a llamarse Jin.** Reemplaza tanto "Yormun" como "Yormungander" en los 6 repos, nombres de paquete (`jin-core`, `jin-executor`), namespaces de K8s (`jin`, `jin-executor`), la clase base de errores (`YormunError` → `JinError`) y los repos de GitHub (`Jin_*`). Se hizo antes de la Fase 7.1 a propósito: nada estaba desplegado todavía, así que el rename fue mecánico y sin downtime. GitHub redirige las URLs viejas, por lo que los links a PRs anteriores en este archivo siguen funcionando.
- **2026-07-31 — Arquitectura de dominios: tres zonas de confianza sobre dos dominios registrables.** `jeanfranck.com` es el portafolio personal del owner (no forma parte de Jin); `jin.jeanfranck.com` es el núcleo confiable (dashboard + API bajo el path `/api`); `jinserver.com` es la zona sandbox de contenido generado por agentes. **La API va como path y no como subdominio** para tener un solo origen: cero CORS y cookie de sesión *host-only* con prefijo `__Host-` en vez de tener que abrirse a `.jeanfranck.com`.
- **2026-07-31 — Se evaluó y descartó consolidar todo en `jeanfranck.com`.** El owner preguntó si se podía prescindir del segundo dominio. Se puede técnicamente (cookies `__Host-` + validación estricta del header `Origin` + CSP cubren los dos ataques principales), pero cambia una garantía impuesta por el navegador por disciplina de código: con dominios separados un bug en la validación de `Origin` queda contenido porque el navegador igual se niega a mandar la cookie cross-site — **falla seguro**; con un solo dominio ese mismo bug deja a una app escrita por un LLM actuando con la sesión del owner, saltándose el HITL — **falla abierto**. Además nunca arregla la reputación en Safe Browsing sobre el dominio del portafolio ni el aislamiento entre previews. El owner compró `jinserver.com`. Se descartó también la variante path-based (`sandbox.jeanfranck.com/<slug>`), que es estrictamente peor: el origen del navegador es esquema+host+puerto y **no** incluye el path, así que todas las previews compartirían un solo origen (service worker de una secuestra a todas, `localStorage` y DOM compartidos).
- **2026-07-31 — El dominio sandbox lleva la marca en el nombre, y se acepta el trade-off.** El patrón canónico (`githubusercontent.com`) usa un nombre deliberadamente no alineado con la marca para no transferirle confianza. `jinserver.com` sí la lleva, lo que debilita solo la dimensión *social* (alguien podría asumir que una preview es contenido oficial); la protección técnica queda intacta al 100% porque sigue siendo un dominio registrable distinto. Se compensa manteniendo la raíz sin contenido y sin branding de Jin ni enlaces de vuelta al panel en las previews. A cambio se gana claridad operativa: dentro de un año se sabrá para qué es y no se dejará vencer por error.
- **2026-07-25 — Fase 4.3: proveedor de embeddings de la memoria — OpenAI `text-embedding-3-large`, no Voyage AI.** BLUEPRINT §6.4 menciona ambas opciones para el corpus en pgvector (no construido todavía), pero no especifica nada para la memoria sqlite-vec — no había ninguna API key de ninguno de los dos en el proyecto. Se le presentaron 3 opciones al owner: reusar Gemini (`GEMINI_API_KEY` ya configurada, cero vendor nuevo), OpenAI, o Voyage AI. Eligió OpenAI explícitamente pese al costo de integración de un vendor nuevo (`OPENAI_API_KEY` nueva, SDK `openai` nuevo) — no la opción de menor fricción. Truncado a 1024 dimensiones (balance calidad/tamaño para un store ≤100k vectores). Vive en `src/memory/embedding-provider.ts`, no en `src/model-provider/` (esa capacidad es TaskProfile/chat-completion con budget guard; embeddings es aparte, sin riesgo "runaway").

## Plan aprobado

0. **Paso previo** — ✅ hecho.
1. **Fase 1.1** [rol Antigravity, ejecuta Claude Code] — ✅ hecho, PR #1 Jin_Infra.
2. **Fase 1.2** [Claude Code] — ✅ hecho, PR #2 Jin_Infra.
3. **Fase 2.1** [rol Antigravity, ejecuta Claude Code] — ✅ hecho, PR #1 en los 4 repos de app.
4. **Fase 2.2** [Claude Code] — ✅ hecho, PR #2 Jin_Core (HITL classifier + audit log). CI verde.
5. **Fase 2.3** [Claude Code] — ✅ hecho, PR #2 Jin_Executor (RBAC + ejecución aislada). CI verde, mergeado.
6. **Fase 3.1** [Antigravity] — ✅ hecho, PR #4 Jin_Core (Canvas LMS + Shadowing Académico). Prerequisitos (`security`/`model-provider`) en PR #3, Claude Code.
7. **Fase 2.4** [Antigravity] — ✅ hecho, PR #5 Jin_Core (bot de Telegram).
8. **Fase 4.1** [Claude Code] — ✅ hecho, PR #6 Jin_Core (Budget guard + kill switch).
9. **Fase 4.2** [Antigravity] — ✅ hecho, PR #9 Jin_Core (Google Calendar + Gmail). Prerequisitos en PR #7 (tools de Calendar en registry.ts) y PR #8 (`ToolExecutorRegistry`/`ApprovalExecutionService`), ambos Claude Code. `canvasScheduleStudyBlock` desbloqueado.
10. **Fase 4.3** [Claude Code] — ✅ hecho, PR #10 Jin_Core (Memoria extendida del agente con sqlite-vec). Ver sección dedicada abajo.

**Fin de la Fase 2, Fase 3.1, Fase 4.1, Fase 4.2 y Fase 4.3 del roadmap.**

**Roadmap extendido (2026-07-25, prompts ya escritos en `docs/PROMPTS.md` — Fase 5+):** el owner pidió planificar el cierre completo del proyecto. Los prompts nuevos asumen las desviaciones reales ya decididas (sin BullMQ, sin Alertmanager, mecanismo aprobar→ejecutar del PR #8) — leer el bloque "Contexto de realidad del código" en PROMPTS.md antes de usarlos:

11. **Fase 5.1** [Claude Code] — ✅ hecho, PR #11 Jin_Core (Agent loop con tool-calling + plan-and-solve + self-correction). Ver sección dedicada abajo. **Era el prerequisito de todo lo demás** — ya desbloqueado.
12. **Fase 5.2** [Claude Code] — ✅ hecho, [Jin_Executor PR #3](https://github.com/JFrnck/Jin_Executor/pull/3) + [Jin_Core PR #12](https://github.com/JFrnck/Jin_Core/pull/12) (`runCode` end-to-end + Modal real). Ver sección dedicada abajo.
13. **Fase 5.3** [Antigravity] — Sesiones reales en Telegram: agent loop + memoria (`src/telegram/**`). Desbloqueado, sin arrancar.
14. **Fase 5.4** [Claude Code] — ✅ hecho, [PR #15](https://github.com/JFrnck/Jin_Core/pull/15) Jin_Core (orquestación multi-agente: sub-agentes coordinados vía task ledger estilo Jira persistido en Postgres). ADR 0005. Ver sección dedicada arriba.
15. **Fase 5.5** [Claude Code] — Pods de servicio: preview apps con puertos expuestos (`npm run dev`, backends) bajo `*.jinserver.com` con TTL (Jin_Executor + Core + Infra, requiere ADR). **Desbloqueado ahora que 5.2 está hecho**, sin arrancar.
16. **Fase 6.1** [Claude Code] — Auth JWT single-user + API REST/WebSocket para interfaces (Jin_Core).
17. **Fase 6.2** [Antigravity] — Web Dashboard MVP (Jin_Web — hoy es el template sin tocar).
18. **Fase 6.3** [Antigravity] — Monaco + Zone Widgets + applets (Jin_Web).
19. **Fase 6.4** [Antigravity] — CLI con Ink (Jin_CLI — hoy es el template sin tocar).
20. **Fase 7.1** [Antigravity] — Deploy real de las apps + Flux GitOps (Jin_Infra + Dockerfiles).
21. **Fase 7.2** [Claude Code] — Runbook de activación real (secretos reales, OAuth consent, webhook Telegram, smoke test E2E).
22. **Fase 7.3** [Claude Code] — Hardening: auditoría de seguridad completa, chaos tests, MCP servers.

Dependencias: 5.1 → (5.2, 5.3, 5.4) → 5.5 (tras 5.2) → 6.1 → (6.2, 6.3, 6.4) → 7.1 → 7.2 → 7.3. Paralelismo natural: Antigravity avanza 5.3 mientras Claude Code hace 5.4/5.5/6.1; Antigravity hace 6.2-6.4 mientras Claude Code prepara 7.2/7.3. **Fase 5.1 y 5.2 hechas — siguiente: 5.3 (Antigravity) y 5.4/5.5 en paralelo (Claude Code elige entre ambas).**

## Bloqueados / esperando

- Ejecución real del bootstrap en la VM OCI la hace el owner (Claude Code solo escribe manifests/scripts).

## Fase 5.1 — resumen técnico (Jin_Core PR #11, mergeado 2026-07-25)

`src/agent/` (BLUEPRINT §6, requisito del owner 2026-07-25: ejecución autónoma orientada a objetivos), área de Claude Code. El agent loop: el LLM ahora puede invocar tools de verdad, con plan-and-solve (2 meta-tools `declarePlan`/`updatePlanStep`, manejadas inline, nunca pasan por HITL) y self-correction (cap de fallos consecutivos por tool+args exactos, no por paso del plan).

**Cambios de contrato en `src/model-provider/`** (afecta a cualquiera que llame `ModelRouterService`/`BudgetGuardedModelRouter`, pero de forma retrocompatible): `ModelMessage.content` ahora acepta `string | bloques[]` (todo caller existente sigue pasando `string` sin cambios), `ModelCompletionResponse` ganó `stopReason` (requerido) y `toolCalls` (opcional). Si en Fase 5.3 tocás código que construye un `ModelCompletionResponse` a mano en un test, va a pedir `stopReason`.

**Contrato de registro para Fase 5.3 (Antigravity):** el agent loop arma su lista de tools desde `listRegisteredTools()` (`src/tools/registry.ts`, ahora con `inputSchema` por tool) y ejecuta/difiere vía `ToolExecutorRegistry` — el mismo registro que ya usás desde Fase 4.2 para `sendEmail`/`deleteCalendarEventFuture` (PR #8). La única diferencia: antes solo importaba para tools `confirm`, ahora el loop también lo usa para invocación INMEDIATA de tools `auto`/`notify`. Para que tus integraciones (Canvas, Google) queden disponibles al agente, cada módulo debe registrar TODAS sus tools (no solo las confirm) en `toolExecutorRegistry.register(nombre, executor)` dentro de su `onModuleInit()` — hoy solo `sendEmail`/`deleteCalendarEventFuture` están registradas; `readEmails`, `listCalendarEvents`, `createCalendarEvent`, etc. todavía no tienen executor, así que si el agente las invoca hoy, falla con `NoExecutorRegisteredError` (fail-safe, no fail-silent).

**Fuera de alcance, documentado:** el plan es transient (no persiste en Postgres — eso es el ledger de Fase 5.4), no hay multi-agente, y no está conectado a Telegram todavía (Fase 5.3).

## Fase 5.2 — resumen técnico (Jin_Executor PR #3 + Jin_Core PR #12, mergeados 2026-07-25)

`runCode` cierra el círculo de BLUEPRINT §4: el agent loop (Fase 5.1) ya puede invocar código real, aislado, con el owner aprobando por Telegram.

**Decisión de diseño clave:** la decisión remoto (Modal) vs. local (pod Deno) ya no es un booleano `remote` que decide el caller (placeholder sin lógica real desde Fase 2.3) — es automática por el Executor según `language: 'typescript' | 'python'` (BLUEPRINT 4.5 lo pide explícito). El tier local es Deno puro, no puede correr Python en absoluto, así que "necesita Python" ya implica "necesita Modal" de forma determinística, sin heurísticas de inspeccionar el código.

**`Jin_Executor` (PR #3):** `ModalService` real con el SDK oficial `modal@0.9.0` — resuelve `App`/imagen de datos científicos (`pandas`/`numpy`) de forma LAZY (cacheada en el primer `runRemote()` real, no en `onModuleInit()` — evita repetir el error de `GoogleOAuthService` en Fase 4.2, que sí escribía a DB de forma eager y rompía `test:e2e`). Sandbox con `blockNetwork`/`outboundDomainAllowlist` según `egressWhitelist` de la tool, límites duros separados del tier local (`remoteMaxTimeoutSeconds: 1800`, `remoteMemoryLimitMiB: 4096` para `runCode`, vs. `maxTimeoutSeconds: 300` local). Egreso: a diferencia del tier local (K3s/Flannel no resuelve dominio→CIDR, ADR 0003 punto 2), la Sandbox API de Modal acepta dominios directos — asimetría documentada en el código para cuando exista una tool con egreso real.

**`Jin_Core` (PR #12):** tool `runCode` declarada en `registry.ts` (`hitlLevel: 'confirm'`) con `inputSchema: {code, language}` — deliberadamente **sin** `env`, defensa en profundidad para que el LLM nunca pueda inyectar variables de entorno arbitrarias al pod/sandbox. Nuevo `src/executor-client/` (área propia de Claude Code, atada a `src/hitl/`, no a `src/integrations/**`): `ExecutorClientService` (HTTP client, `EXECUTOR_BASE_URL` requerida fail-fast) + `ExecutorClientModule` que registra el ejecutor de `runCode` en `ToolExecutorRegistry` al bootstrapear — mismo patrón que `GoogleModule` (Fase 4.2). `env` siempre se envía `{}` al Executor.

**Criterio de éxito cerrado:** turno del agente "analiza este CSV con pandas" → LLM invoca `runCode({code, language: 'python'})` → clasifica `confirm` → pending approval → owner aprueba por Telegram → `ApprovalExecutionService` → `ToolExecutorRegistry` → `ExecutorClientService` → Executor real → Modal → resultado (stdout/stderr) sanitizado con `wrapUntrustedContent` (ya lo hace el agent loop para todo resultado de tool, Fase 5.1) vuelve al chat.

**Fuera de alcance, documentado:** `runCode` con TypeScript local ya funcionaba desde Fase 2.3 (pod Deno) — esta fase solo conecta a Core y agrega el tier Modal. No hay preview services de larga vida todavía (eso es Fase 5.5, ahora desbloqueada).

## Fase 4.3 — resumen técnico (Jin_Core PR #10, mergeado 2026-07-25)

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

## Fase 4.2 — decisiones y prerequisitos (Jin_Core PR #7)

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

## Fase 2.3 (follow-up) — RBAC de NetworkPolicy en Jin_Infra (PR #4)

Cierra ADR 0003 punto 3. `k8s/base/executor/`: `ServiceAccount` `executor` en `jin-executor`, `Role` `executor-agents-sandbox` en `agents-sandbox` (`create/get/list/delete` de Pods per BLUEPRINT 4.2 + `create/delete` de NetworkPolicies, la extensión que ADR 0003 identificó como necesaria), `RoleBinding` conectando ambos across namespaces. Sin acceso a Secrets/ConfigMaps/Deployments/otros namespaces. Verificado con `kubectl kustomize k8s/base` + CI verde (lint manifests, shellcheck, bats, GitGuardian).

## Fase 2.3 — resumen técnico (Jin_Executor PR #2)

Implementado según ADR 0003 (separación de privilegios + hallazgo de seguridad `deno eval`):

- `src/rbac/` — whitelist propia (`runCode`, sin egreso), `RbacValidatorService` como única puerta de entrada (403 si no está en whitelist).
- `src/k8s/k8s.service.ts` — único punto de contacto con `@kubernetes/client-node` en todo el proyecto; `namespace` fijo al construir, estructuralmente no puede tocar otro namespace.
- **Hallazgo de seguridad real:** verificado contra Deno 2.9.3 real que `deno eval` tiene *"implicit access to all permissions"* — ignora `--allow-net` completamente. Corregido a `deno run` + código como `data:` URL en base64 (sin ConfigMap, sin shell).
- `src/pod-lifecycle/` — ciclo de vida bajo demanda, sin warm pool, siempre destruye el pod.
- `src/modal/` — stub explícito (501) para `remote: true`, Fase 5.
- **Tests:** 30 unitarios + 5 e2e (capa HTTP completa sin K8s real) + 2 de integración con **K3s real** (`@testcontainers/k3s`): camino feliz de ejecución, y el test de aislamiento de red real que pide PROMPTS.md (un pod en `agents-sandbox` con permiso de red de Deno explícito igual no alcanza un servicio en `jin`, bloqueado por la NetworkPolicy real).
- **Fix incidental:** `app.controller.ts` (heredado de Fase 2.1) tenía un endpoint stub `@Post('execute')` que colisionaba de ruta con el `ExecuteController` real — toda petición a `/execute` devolvía 201 del stub, saltándose RBAC/Zod por completo. Corregido.

## Fase 2.2 — resumen técnico (Jin_Core PR #2)

Implementado según ADR 0001 (4 niveles HITL) y ADR 0002 (`request_id` + `pending_approvals`):

- `src/tools/registry.ts` — 3 tools stub con hitlLevel estático, `Object.freeze()`-ado.
- `src/hitl/` — `classifier.ts` (nunca decide por `inputs`, tool desconocida → `UnknownToolError`), `dual-confirm.service.ts` (estado persistido en Postgres, 30s reales entre aprobaciones), `timeout.service.ts` (nunca aprueba; descarta o escala/abandona).
- `src/audit/` — `hash-chain.ts` (puro), `audit.service.ts` (insert-only, advisory lock para serializar escrituras concurrentes entre réplicas durante rolling update), `chain-verification.service.ts` (cron diario, bloquea escrituras si detecta corrupción).
- **Infra nueva que no existía:** Drizzle + `pg` (primera vez que Core habla con Postgres), migraciones con `.down.sql` a mano, `src/config/` (Zod + `@nestjs/config`, fail-fast).
- **Tests:** 36 unitarios (100% matriz tool×nivel, 4 tests de mutación del hash chain) + 18 de integración con testcontainers (Postgres real) + 1 e2e.
- **Gap de cobertura documentado:** `timeout.service.ts` en ~70% — los caminos 'escalate'/'abandon' no tienen integration test porque ninguna tool registrada usa `timeoutBehavior: 'escalate'` todavía (llega con Canvas, Fase 3).

## CI de Jin_Core / Jin_Executor: bugs encontrados y corregidos (2026-07-21)

El owner reportó que el CI de ambos repos falló al pushear la Fase 2.1. Investigación encontró **cuatro problemas reales**, todos corregidos:

1. **Causa raíz del fallo de CI:** pnpm 11 bloquea por defecto scripts de instalación no aprobados (`ERR_PNPM_IGNORED_BUILDS`). Fix: `pnpm-workspace.yaml` con `allowBuilds`.
2. **Grave, independiente del CI:** `node_modules/` (289M) y `dist/` estaban commiteados en git en ambos repos. Corregido: `.gitignore` estándar + `git rm --cached`.
3. **Bug funcional real en Executor:** `app.controller.ts` importaba `AppService` desde el archivo equivocado y tipaba la dependencia como `any` — rompía el DI de Nest en runtime.
4. **`@typescript-eslint/no-explicit-any` estaba en `'off'`** en el scaffolding — contradice AGENTS.md 3.2. Reactivado en `'error'`.

También corregido de paso: glob patterns rotos en `lint`/`format`, y `package.json` `name` de ambos repos.

**CI verificado en verde** en Core PR #1/#2 y Executor PR #1; Executor PR #2 esperando confirmación final.

## Recientemente completado (últimos 7 días)

- 2026-07-31: **Rename global Yormun/Yormungander → Jin, completo en los 6 repos.** PRs mergeados con CI verde: [Core #13](https://github.com/JFrnck/Jin_Core/pull/13), [Executor #4](https://github.com/JFrnck/Jin_Executor/pull/4), [Infra #5](https://github.com/JFrnck/Jin_Infra/pull/5), [Web #2](https://github.com/JFrnck/Jin_Web/pull/2), [CLI #2](https://github.com/JFrnck/Jin_CLI/pull/2), más el commit de docs `56b3eb5`. Incluye paquetes (`jin-core`, `jin-executor`, y de paso `jin-web`/`jin-cli`, saldando la deuda de `temp-*` pendiente desde la Fase 2.1), clase base de errores `JinError` (+ `JinErrorFilter`), namespaces de K8s (`jin`, `jin-executor`), imagen de Modal `jin-data-science`, bucket R2 `jin-backups`, lock advisory de la audit chain, y el remapeo de dominios a `jin.jeanfranck.com` + `jinserver.com`. Los 6 repos renombrados en GitHub (`Jin_*`) y las carpetas locales bajo `Jin/`; GitHub redirige las URLs viejas con 301, así que los links a PRs anteriores de este archivo siguen funcionando (verificado con `curl`). **Verificación del aislamiento**, que era el riesgo real del cambio de namespaces: `kubectl kustomize` renderiza los 60 recursos, los 4 namespaces se declaran, las 12 NetworkPolicies apuntan a namespaces existentes y los `namespaceSelector` resuelven — más el test de integración contra K3s real, que sigue probando que un pod en `agents-sandbox` no alcanza un servicio en `jin`. Un rename a medias ahí habría roto el aislamiento en silencio, sin fallar ruidosamente.
- 2026-07-25: [Jin_Executor] [PR #3](https://github.com/JFrnck/Jin_Executor/pull/3) mergeado + [Jin_Core] [PR #12](https://github.com/JFrnck/Jin_Core/pull/12) mergeado — Fase 5.2 completa en ambos repos: `ModalService` real (SDK `modal@0.9.0`, sandbox Python/pandas con imagen cacheada de forma lazy), ruteo local/remoto automático por `language` (reemplaza el `remote: boolean` placeholder de Fase 2.3), tool `runCode` declarada en `registry.ts` (confirm, sin `env` expuesto al LLM) y `src/executor-client/` nuevo conectando el agent loop al Executor real. 51 tests nuevos entre ambos repos (42 unitarios + 2 integración K3s real + 5 e2e en Executor; 258 unitarios + 43 integración Postgres real + 1 e2e en Core tras el merge), CI verde verificado con `gh pr checks` en ambos. Ver sección dedicada abajo.
- 2026-07-24: [Jin_Core] [PR #7](https://github.com/JFrnck/Jin_Core/pull/7) mergeado — prerequisito de Fase 4.2: 4 tools de Calendar declaradas en `registry.ts` (`listCalendarEvents` auto, `updateCalendarEvent` notify, `deleteCalendarEventPast` notify, `deleteCalendarEventFuture` confirm). Matriz HITL actualizada a 10 tools, 100% cobertura.
- 2026-07-24: [Jin_Core] [PR #6](https://github.com/JFrnck/Jin_Core/pull/6) mergeado — Fase 4.1 completa: `src/budget/` (tracking sesión/día, degradación al 80% vía el hint `budgetRemaining` de Fase 3.1, kill switch de runaway persistido en Postgres), métricas Prometheus, `BudgetGuardedModelRouter` ya inyectado en Canvas y Telegram, `/budget` con datos reales y `/unpause` nuevo en el bot. 87 tests nuevos (unitarios + integración Postgres real), CI verde, verificado con `tsc --noEmit -p tsconfig.json` de forma independiente antes de mergear.
- 2026-07-24: [Jin_Core] [PR #5](https://github.com/JFrnck/Jin_Core/pull/5) mergeado — Fase 2.4 completa: bot de Telegram con grammY en modo webhook, auth estricta con `TELEGRAM_OWNER_CHAT_ID` numérico, `TELEGRAM_WEBHOOK_SECRET` requerida fail-fast validando el header de Telegram, `bot.init()` real, comandos `/start`/`/status`/`/tasks`/`/approve`/`/reject`/`/budget` (stub honesto), integración con `DualConfirmService`/`AuditService`/`ModelRouterService`. 3 rondas de review contra el código real (bug de tipos chat_id, gap de seguridad del webhook, `botInfo` hardcodeado, tipos en specs invisibles a `pnpm build`), todas resueltas y verificadas independientemente antes de mergear. 105 tests en verde.
- 2026-07-23: [Jin_Core] [PR #4](https://github.com/JFrnck/Jin_Core/pull/4) mergeado — Fase 3.1 completada por Antigravity (integración Canvas LMS, cliente REST con rate limit de 30 req/min, handlers sanitizados con `wrapUntrustedContent`, ShadowingService nocturno consumiendo `ModelRouterService.complete('long_context', ...)` con Gemini 3.1 Pro, `CalendarNotImplementedError` 501). 95 tests en verde. Claude Code verificó de forma independiente (lint/test/build/typecheck a mano, no solo el self-report) antes de mergear — sin hallazgos nuevos.
- 2026-07-23: [Jin_Core] PR #3 mergeado — prerequisitos de Fase 3.1: `src/security/injection-sanitizer.ts` (ADR 0004) + `src/model-provider/**` (router, failover, providers Anthropic/Google, config/models.yaml) + declaración de las 3 tools de Canvas en `registry.ts`. 87 tests, cobertura 97% en los módulos nuevos. Fix incidental de CI (env vars faltantes). Antigravity desbloqueado para retomar `integrations/canvas/**`.
- 2026-07-23: [Jin_Infra] PR #4 mergeado — RBAC de NetworkPolicy para el ServiceAccount del Executor (ADR 0003 punto 3), cerrando el último follow-up técnico de la Fase 2.
- 2026-07-23: Review y merge de los 8 PRs de las Fases 1-2 en los 6 repos. Incidente: Jin_Infra #2 auto-cerrado por GitHub al borrar la rama base de un PR apilado; recuperado como #3. Procedimiento corregido aplicado sin incidentes en Core y Executor. Todos los repos en `main` limpio, 0 PRs abiertos.
- 2026-07-22: [Jin_Executor] Fase 2.3 completa (RBAC + ejecución aislada con K3s real). PR #2 abierto. 30 unitarios + 5 e2e + 2 de integración (K3s real vía testcontainers).
- 2026-07-22: [Jin_Docs] ADR 0003 (Executor: separación + hallazgo `deno eval`) escrito.
- 2026-07-21: [Jin_Core] Fase 2.2 completa (HITL classifier + audit log + Drizzle + config module). PR #2 abierto, CI verde.
- 2026-07-21: [Jin_Docs] ADR 0001 (4 niveles HITL) y ADR 0002 (`request_id`/`pending_approvals`) escritos; BLUEPRINT 9.5 corregido.
- 2026-07-21: [Jin_Core, Jin_Executor] Migración Jest→Vitest + fix de CI + fix de node_modules/dist trackeados + fix de bug de DI en Executor + reactivación de `no-explicit-any` + fix de globs de lint/format + rename de `package.json`.
- 2026-07-21: PRs abiertos en los 4 repos de app para la Fase 2.1.
- 2026-07-21: [Jin_Infra] PR #2 abierto (Fase 1.2, backups).
- 2026-07-20: [Jin_Infra] Fase 1.1 (infra base) terminada; PR #1 abierto.
- 2026-07-20: [Jin_Core, Jin_Executor, Jin_Web, Jin_CLI] Fase 2.1 completada por Antigravity (scaffolding base de apps y CI pipelines), commiteada en local.
- 2026-07-19: [Jin_Docs] Bundle de documentación inicial commiteado (`main`).
- 2026-07-19: [todos los repos] Repos creados con README inicial; stubs AGENTS.md/CLAUDE.md colocados.
