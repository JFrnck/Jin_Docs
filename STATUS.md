# STATUS

## Última actualización: 2026-08-04 (America/Lima) — actualización 23

> **Antigravity ya está activo** (ver abajo, Fase 3.1/2.4/4.2). Retoma ownership normal de `docs/WORKFLOW.md` sección 2 — Claude Code ya no asume tareas `[ANTIGRAVITY]` por defecto, salvo negociación puntual vía esta misma nota.

## En progreso

### Claude Code

- **Repo:** `Jin_Core`. **Recomendación #1** (readiness/liveness probes) implementada — [PR #22](https://github.com/JFrnck/Jin_Core/pull/22). **Recomendación #2** (poda + compresión del historial de chat) implementada — [PR #23](https://github.com/JFrnck/Jin_Core/pull/23), ver sección dedicada abajo.
- **Próximo:** sin tarea propia asignada. Ambos P0 de la Ronda 1 de `docs/RECOMENDACIONES.md` cerrados; PRs #22 y #23 esperando revisión/merge del owner. **Nueva: Ronda 2 de recomendaciones** (auditoría cross-repo de los 6 repos, 2026-08-05) — 18 puntos nuevos sin implementar, ver abajo y el documento completo. Fase 7.1 (deploy real) sigue siendo de Antigravity.
- **Handoffs abiertos hacia Antigravity (`Jin_CLI`):** (a) adoptar `compactedHistory` o el PR #23 queda sin consumidor real; (b) **regenerar `source/api-types.ts`** — está desincronizado desde el PR #21 y por eso `jin tasks` no muestra `actor`/`externalInputsSummary`, ver Recomendación 20.b.

### Antigravity

- **Repo:** `Jin_CLI`
- **Descripción:** Fase 6.2 (CLI real en `Jin_CLI`) completa — [PR #3](https://github.com/JFrnck/Jin_CLI/pull/3), CI verde.
- **Detalle de implementación:**
  - `Jin_CLI` implementada usando **Ink v4**, **`openapi-fetch`**, **`socket.io-client`** y tipos TypeScript auto-generados con `openapi-typescript` desde `Jin_Core/contracts/openapi.json` (`pnpm generate:api`). Cero tipos copiados a mano.
  - Almacenamiento seguro de credenciales Bearer JWT en `~/.config/jin/auth.json` con permisos POSIX `0600` (`fs.chmodSync(path, 0o600)`), garantizando acceso exclusivo al usuario del proceso OS.
  - Comandos CLI completos: `jin login`, `jin status`, `jin tasks`, `jin approve <id>`, `jin reject <id>`, `jin chat`, y `jin memory <query>`.
  - Continuidad de sesión interactiva en `jin chat` manteniendo `sessionId` (UUID) y acumulando `history: ModelMessage[]` en memoria del cliente Ink por turno.
  - Manejo de estados de carga ("pensando...") para ejecuciones no-streaming del agente, captura de evento `disconnect` en Socket.IO para notificar tokens expirados, manejo de HTTP 429 (@Throttle 5/15m) en login, y manejo exhaustivo de outcomes HITL (`resolved`, `awaiting-second`, HTTP 409 `SecondApprovalTooEarlyError` tras confirmaciones <30s, HTTP 404).
  - Suite de pruebas unitarias en AVA + `ink-testing-library` con Prettier y XO limpios (6/6 tests pasando, build TypeScript limpio, CI de GitHub Actions verde en `Jin_CLI`).

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
| Jin_Core | [#14](https://github.com/JFrnck/Jin_Core/pull/14) | Fase 5.3: sesiones reales en Telegram (agent loop + memoria + 8 executors client-level) | ✅ mergeado |
| Jin_Core | [#15](https://github.com/JFrnck/Jin_Core/pull/15) | Fase 5.4: orquestación multi-agente (task ledger persistido, `OrchestratorService`) | ✅ mergeado |
| Jin_Executor | [#5](https://github.com/JFrnck/Jin_Executor/pull/5) | Fase 5.5: `PreviewServiceLifecycleService`, pods de servicio bajo `*.jinserver.com` | ✅ mergeado |
| Jin_Core | [#16](https://github.com/JFrnck/Jin_Core/pull/16) | Fase 5.5: tools `startPreviewService`/`stopPreviewService`/`listPreviewServices` | ✅ mergeado |
| Jin_Infra | [#6](https://github.com/JFrnck/Jin_Infra/pull/6) | Fase 5.5: RBAC + Certificate + PVC para pods de servicio | ✅ mergeado |
| Jin_Core | [#17](https://github.com/JFrnck/Jin_Core/pull/17) | Fase 6.1: Auth JWT single-user + API REST/WebSocket | ✅ mergeado |
| Jin_Core | [#18](https://github.com/JFrnck/Jin_Core/pull/18) | Fase 6.1.1: datos reales de budget + contrato OpenAPI completo (nestjs-zod) | ✅ mergeado |
| Jin_Core | [#19](https://github.com/JFrnck/Jin_Core/pull/19) | Fase 6.1.2: endpoints REST de orquestación + preview services | ✅ mergeado |
| Jin_Web | [#3](https://github.com/JFrnck/Jin_Web/pull/3) | Fase 6.2+6.3: Dashboard completo (10 pantallas, no MVP) | ✅ mergeado |

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
- **2026-08-02 — `Jin_Web` (dashboard) pasa de Antigravity a Claude Code.** El owner pidió explícitamente que todo lo relativo a la web lo lidere Claude Code de ahora en adelante ("confío más en ti con lo que respecta a web"). A diferencia de las demás decisiones de esta sección, esta NO nace de una desviación técnica encontrada en el código — es una reasignación de ownership por preferencia directa del owner, contraria al criterio de "dividir por fortaleza" que motivó la asignación original a Antigravity (`docs/WORKFLOW.md` §1, frontend grande se beneficia del contexto amplio de Gemini). Actualizado: `docs/WORKFLOW.md` §2.1 (tabla + nota de excepción) y §6.3, `AGENTS.md` §7.4, `CLAUDE.md` §3, `README.md` (tabla de repos), `docs/PROMPTS.md` (6.2/6.3 re-etiquetados `[CLAUDE CODE]`, nota de dependencias), y los stubs `CLAUDE.md`/`AGENTS.md` dentro del propio repo `Jin_Web`. `Jin_CLI` e `Jin_Infra` siguen con Antigravity — la reasignación es específica a `Jin_Web`. Ver la estructura propuesta del dashboard en la sesión donde se tomó esta decisión (no persistida aparte; Claude Code la retoma al iniciar la Fase 6.2).

- **2026-08-02 — El dashboard es una PWA instalable. BLUEPRINT §8.1.1 nuevo.** El owner pidió estructurar el dashboard como PWA para generar el diseño en Claude Design. No estaba en el blueprint (§8.1 solo decía "static build en Cloudflare Pages"), así que se formalizó en vez de dejar la divergencia abierta (regla de oro #12). El valor real no es offline sino **instalable en el móvil**: hoy aprobar desde el celular solo se puede por Telegram. Se fijaron 3 reglas duras porque un PWA mal configurado degrada la seguridad: (a) el service worker **jamás** cachea `/api/*` — un `runtimeCaching` genérico persistiría aprobaciones, payloads de correos y audit log en `CacheStorage`, sobreviviendo al logout (borrar la cookie no borra el cache); (b) offline es solo lectura, **prohibido encolar mutaciones** — una aprobación sincronizada 3 h tarde decide sobre estado vencido o con el TTL de 24 h ya expirado, el HITL asume decisión presente sobre estado presente; (c) el SW vive solo en el origen confiable, garantizado ya por la separación de dominios de §5.2. **Push notifications fuera de alcance**: requieren VAPID keys, persistencia de suscripciones y lógica de envío en `jin-core`, nada construido — cuando existan, reemplazan el canal de Telegram, no lo duplican. Reglas replicadas en `Jin_Web/AGENTS.md` (punto de lectura real de quien implemente).
- **2026-08-02 — Estado del dashboard: TanStack Query sí, Zustand no (todavía).** El owner preguntó si convenía dejar Zustand implementado de entrada para no desarrollarlo después. Se recomendó no: casi todo el estado del dashboard es estado *de servidor* (pendientes, audit, budget, chat), que cubre TanStack Query + invalidación por WS; y Zustand es la librería de estado con **menor costo de migración** del ecosistema (un store es un hook — sin Provider, sin context boundary, sin boilerplate de actions), así que el argumento "hay que dejarlo listo para no rehacerlo" no aplica como aplicaría a Redux. El costo de meterlo vacío sí es real: crear un lugar donde el estado *puede* vivir invita a que alguien meta `pendingApprovals` ahí, generando dos fuentes de verdad contra el cache de Query. **Regla pre-aprobada para no re-litigarlo:** Zustand entra cuando aparezca estado de cliente que (a) no sea estado de servidor y (b) lo compartan dos componentes sin relación padre-hijo. Candidato real: el estado del editor en Fase 6.3 (documento abierto, widgets activos, sugerencias pendientes entre Monaco y el panel lateral). Si al planificar 6.3 se ve así, entra con justificación escrita y sin volver a discutir la decisión.
- **2026-08-03 — Adoptar `nestjs-zod` para que el contrato OpenAPI documente request/response de verdad.** Revisando el diseño v2 del dashboard contra la API real se encontró que ningún endpoint de Fase 6.1 estaba documentado en `contracts/openapi.json` (0 de 10) — interfaces TS planas sin decoradores no son introspectables por `@nestjs/swagger`, así que `pnpm generate:api` en Jin_Web habría generado tipos vacíos para todo. Se presentaron 2 opciones: DTOs a mano con `@ApiProperty` (cero dependencia nueva, pero duplica cada shape — el Zod schema que ya valida en runtime, más una clase con decoradores solo para Swagger) o `nestjs-zod` (una sola fuente de verdad: el schema Zod sirve para validación + tipo + doc). El owner eligió `nestjs-zod` explícitamente pese a ser una dependencia nueva — verificado antes de recomendarlo que soporta exactamente el stack real (`zod@4`, `@nestjs/swagger@11`, `@nestjs/common@11`). Reemplaza el `ZodValidationPipe` casero (borrado). Detalle técnico y el bug real de `bigint` en `AuditLogRow.id` encontrado en el proceso: ver [PR #18](https://github.com/JFrnck/Jin_Core/pull/18) y la sección "En progreso" de Claude Code.

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
13. **Fase 5.3** [Antigravity] — ✅ hecho, [PR #14](https://github.com/JFrnck/Jin_Core/pull/14) Jin_Core (sesiones reales en Telegram: agent loop + memoria). Ver sección dedicada arriba.
14. **Fase 5.4** [Claude Code] — ✅ hecho, [PR #15](https://github.com/JFrnck/Jin_Core/pull/15) Jin_Core (orquestación multi-agente: sub-agentes coordinados vía task ledger estilo Jira persistido en Postgres). ADR 0005. Ver sección dedicada arriba.
15. **Fase 5.5** [Claude Code] — ✅ hecho, [Jin_Executor PR #5](https://github.com/JFrnck/Jin_Executor/pull/5) + [Jin_Core PR #16](https://github.com/JFrnck/Jin_Core/pull/16) + [Jin_Infra PR #6](https://github.com/JFrnck/Jin_Infra/pull/6) (pods de servicio: preview apps con puertos expuestos bajo `*.jinserver.com`, TTL obligatorio). ADR 0006. Ver sección dedicada arriba.
16. **Fase 6.1** [Claude Code] — ✅ hecho, [PR #17](https://github.com/JFrnck/Jin_Core/pull/17) Jin_Core (Auth JWT single-user + API REST/WebSocket). ADR 0007. Ver sección dedicada arriba.
17. **Fase 6.2** [Claude Code] — ✅ hecho, [PR #3](https://github.com/JFrnck/Jin_Web/pull/3) Jin_Web (Web Dashboard completo, no MVP). Reasignado de Antigravity el 2026-08-02. Ver sección dedicada arriba.
18. **Fase 6.3** [Claude Code] — ✅ hecho, mismo [PR #3](https://github.com/JFrnck/Jin_Web/pull/3) (Monaco + Zone Widgets + preview apps — construidas junto con 6.2 a pedido del owner, no en dos PRs separados).
19. **Fase 6.4** [Antigravity] — ✅ hecho, [PR #3](https://github.com/JFrnck/Jin_CLI/pull/3) Jin_CLI (CLI con Ink: login/status/tasks/approve/reject/chat/memory). Revisada por Claude Code en 3 rondas antes de implementar (ver sección dedicada abajo) + [PR #4](https://github.com/JFrnck/Jin_CLI/pull/4) (fix de falso positivo de GitGuardian).
20. **Fase 7.1** [Antigravity] — Deploy real de las apps + Flux GitOps (Jin_Infra + Dockerfiles). **Siguiente en la cola.**
21. **Fase 8.1** [Claude Code] — Infisical SDK en runtime (Core + Executor). **Va antes de 7.2** — ver auditoría abajo.
22. **Fase 7.2** [Claude Code] — Runbook de activación real (secretos reales, OAuth consent, webhook Telegram, smoke test E2E).
23. **Fase 8.2** [Claude Code] — Golden set de prompt injection (~50 adversariales, BLUEPRINT §13.1 obligatorio). **Va antes de 7.3.**
24. **Fase 7.3** [Claude Code] — Hardening: auditoría de seguridad completa, chaos tests, MCP servers.
25. **Fase 9.1** [Antigravity] — Notion + comando `/audio` (transcripción).
26. **Fase 9.2** [Claude Code] — GitHub App + capacidad git real (desbloquea `mergeAgentBranch`, hoy 501).
27. **Fase 9.3** [Claude Code] — RAG de corpus propio en pgvector (correos/PDFs/notas — la otra mitad de BLUEPRINT §3.3.1).
28. **Fase 9.4** [Antigravity] — Alerta matutina 06:00 con resumen de prioridades.
29. **Fase 9.5** [Claude Code] — Feature flags en caliente (ConfigMap + SIGHUP).
30. **Fase 9.6** [Antigravity] — Comandos de administración en la CLI + menús navegables.

Dependencias: 5.1 → (5.2, 5.3, 5.4) → 5.5 (tras 5.2) → 6.1 → (6.2+6.3 en secuencia [Claude Code], 6.4 [Antigravity] en paralelo) → **7.1 → 8.1 → 7.2 → 8.2 → 7.3 → Fase 9 (cualquier orden, ninguna bloquea a otra)**. **Fases 1-6 completas. Siguiente: Antigravity ejecuta 7.1 (deploy real); Claude Code queda a la espera de que exista despliegue para 8.1/7.2, o puede adelantar cualquier fase 9 en paralelo si el owner lo prefiere.**

**Nota de numeración:** el número es una etiqueta, no un orden de ejecución — 8.1 corre antes que 7.2 a propósito (ver auditoría). Ya hay precedente en este roadmap: la Fase 3.1 se ejecutó antes que la 2.4.

## Fase 0.1 — debug #1 (2026-08-05, en curso)

El owner pidió corregir los 18 puntos de la Ronda 2 de `docs/RECOMENDACIONES.md` (10-26 + 20.b). Numerada "0.1" a propósito — es deuda encontrada, no una fase nueva del producto; no reordena el roadmap de arriba, corre en paralelo. Split por dueño real de cada repo (`docs/WORKFLOW.md` §2):

**[Claude Code] — en ejecución esta sesión:**
- [ ] 10 + 26 — Jin_Executor: zip-slip en `files` de `startPreviewService` + `readOnlyRootFilesystem` + límites del init container.
- [ ] 11 + 12 + 13 — Jin_Core (`src/hitl`, `src/audit`): 3 notificaciones muertas → Telegram real, lock de audit chain persistido, `audit_log` conserva actor/inputs externos al resolver.
- [ ] 16 (mitad Web) + 17 — Jin_Web: manejo de `disconnect` del WS, mutaciones de `ApprovalCard`/`preview`/`budget`/`editor` sin `catch`.
- [ ] 18 — Jin_Core: bump `js-yaml` (advisory alto, dependencia de producción).
- [ ] 19 — Jin_Web: `monaco-editor`→`dompurify` vulnerable, evaluar upgrade.
- [ ] 3 + 4 + 5 (Ronda 1, seguían sin tocar) — Jin_Core: `.env` en scripts standalone, colisión Redis e2e, heap de Node en tooling.

**[Antigravity] — handoff, no se toca desde esta sesión sin negociación previa (`Jin_CLI/CLAUDE.md`):**
- [ ] 20.b — regenerar `Jin_CLI/source/api-types.ts` (desincronizado desde el PR #21 de Core) + mostrar `actor`/`externalInputsSummary` en `TasksView`.
- [ ] 14 — `jin login` deja de tomar la contraseña como argumento posicional (prompt enmascarado).
- [ ] 16 (mitad CLI) — manejo de `disconnect` del WS + reconexión en `ws-chat.ts`.
- [ ] 20 (mitad CLI) — sacar `|| true` de `pnpm run test`/`generate:api` en el CI de Jin_CLI, agregar `lint`.
- [ ] 15 — Jin_Infra: `verify-restore.sh` extendido a Redis y `memory.db`, no solo Postgres.
- [ ] 25 — Jin_Infra: pinnear checksum/firma de los instaladores de K3s/Flux en bootstrap.

**[Claude Code, cuando exista quien lo priorice] — deuda de fondo, no bloqueante:**
- [ ] 21 — specs faltantes en `chain-verification.service.ts`, `ws-token.ts`, gateways WS, `env.schema.ts`.
- [ ] 22 — Jin_Web sin ningún test unitario (más allá del gap de e2e ya conocido).
- [ ] 23 — Jin_CLI, cobertura mínima.
- [ ] 24 — Jin_Infra: cablear `readinessProbe`/`livenessProbe` a `/health/ready`/`/health/live` cuando exista el Deployment real (Fase 7.1).

Se irá tachando y anotando PR por punto a medida que se cierre.

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

## Fase 6.1.2 — resumen técnico (Jin_Core PR #19, mergeado 2026-08-03)

Cierre del último gap de API antes de construir el dashboard completo: el board de orquestación (pantalla 4.8 del brief) y las preview apps (4.10) no tenían endpoint REST, solo existían como tools que invoca el agente.

- `GET /api/orchestrator/runs` (paginado, cursor por `createdAt` — `agentOrchestrationRuns.id` es uuid, sin orden natural, mismo criterio de `limit+1` que `AuditService.listRecent`) y `GET /api/orchestrator/runs/:runId` (run + tickets + hilo de comentarios de cada uno en una sola llamada — 404 real vía `RunNotFoundError`/`JinErrorFilter`). `LedgerRepository` ganó `listRuns()`/`getRun()`.
- `GET /api/preview-services` / `DELETE /api/preview-services/:id` — delegan directo en `ExecutorClientService.listPreviewServices()`/`stopPreviewService()`, ya usados por el agent loop; sin lógica nueva.
- Todos los DTOs vía `nestjs-zod` desde el día uno (`createZodDto` + `@ZodResponse` + `dateCodec`), evitando repetir el gap de contrato vacío de Fase 6.1.

Verificado: 330 unitarios + 68 integración (Postgres real) + 19 e2e (pipeline real, incluido el 404), `gh pr checks` real en verde antes de mergear.

## Fase 6.2+6.3 — resumen técnico (Jin_Web PR #3, mergeado 2026-08-04)

El owner pidió explícitamente web **completa, no un MVP recortado** — las 10 pantallas del diseño v3 aprobado (`docs/WEB_DESIGN_BRIEF.md`), no solo las 6 de PROMPTS.md §6.2. Esto empujó primero la Fase 6.1.1 (arriba) para que el board de orquestación y las preview apps tuvieran datos reales, y recién después este PR.

**Scaffolding real, reemplazando el template de Vite.** React Router **v8** (no v7 — BLUEPRINT 8.1 quedó desactualizado en la versión exacta, corregido: v8.3.0 es la evolución directa, mismas convenciones de framework mode, peer deps exactas con `react@19.2.7`/`vite@8`). SPA mode (`ssr:false`) para build estático a Cloudflare Pages. `resolve: { tsconfigPaths: true }` en `vite.config.ts` — sin esto el alias `~/*` no resuelve en dev (encontrado al primer arranque real del dev server, no en build).

**Las 10 pantallas:** login, overview, bandeja HITL (`ApprovalCard` con los 4 niveles de riesgo, timer de dual-confirm en vivo cada segundo), chat (WS + plan en progreso), audit log (paginado por cursor), presupuesto + kill switch (`unpause` con hold-to-confirm de 3s, igual al diseño — nunca una tool, nunca en la bandeja), memoria (búsqueda semántica), board de orquestación (columnas por status + panel de conflicto que linkea a la bandeja real en vez de fabricar botones de aprobación sin `requestId`), editor Monaco con Zone Widgets reales (`changeViewZones`, no un tooltip), preview apps.

**Seguridad, aplicada tal como se decidió en el plan:** `api-client.ts` nunca toca el JWT — cookie `__Host-jin_session` vía `credentials:'include'`, mismo criterio en el WS (`withCredentials:true`, coincide con `extractWsToken` de Jin_Core). Un 401 en cualquier momento (no solo al cargar) dispara `location.assign('/login?returnTo=...')` — cubre la "sesión expirada a mitad de una decisión" del diseño §07 con una sola pieza de código, no una por pantalla.

**Paleta de riesgo validada con el skill de dataviz, no elegida a ojo.** Los 4 niveles HITL son ordinales (auto < notify < confirm < dual-confirm), no categóricos — se validaron con `validate_palette.js --ordinal` (un ramp de un solo matiz, monótono en lightness) contra la superficie real `#08090B`, no el surface de referencia del skill. `dual-confirm` = `#E2543F` (el accent ya aprobado en el diseño); `notify`/`confirm` se derivaron del mismo matiz (búsqueda automatizada de la combinación que pasa las 4 chequeas) para que el accent quedara reservado, como pedía el diseño.

**Bug real encontrado verificando en navegador (Playwright headless), no solo build/lint:** `openapi-fetch` podía resolver `{ data: undefined, error: undefined }` ante un fallo de red real (proxy caído) — TanStack Query no tolera un `queryFn` que resuelve `undefined` y la pantalla de `/hitl` quedaba **completamente en negro** en vez de mostrar un error. Se centralizó en `unwrap()` (`api-client.ts`, usado por los 8 hooks de datos) y se agregó manejo de `isError` explícito donde faltaba (`budget`, `audit`, `orchestrator`, `orchestrator/:runId`, `preview`, `memory` — antes mostraban skeleton infinito o un "vacío" que en realidad era un error de red). Ninguno de los dos bugs aparece en `tsc`/lint/build — solo se manifiestan en el pipeline real request→fetch→React Query, exactamente el mismo motivo por el que Fase 6.1.1 encontró el bug de `bigint` solo vía e2e real.

**Verificación:** `pnpm typecheck` (`react-router typegen && tsc --noEmit`) limpio, `pnpm lint` limpio (1 warning esperado, mismo patrón que el template oficial de React Router), `pnpm build` limpio (SPA mode). Navegador real (Playwright headless, Chromium instalado en la sesión): las 10 pantallas + vista móvil (nav inferior de 4 ítems: Overview/Aprobar/Chat/Gasto) sin errores de consola tras el fix. CI real verde (`build` 19s). **No verificado:** flujo end-to-end contra `Jin_Core` corriendo local con Postgres/Redis reales y datos genuinos — esta sesión no levantó el backend completo, solo el dev server de Jin_Web contra un proxy sin destino.

**Gaps reales encontrados y documentados, sin rellenar con datos falsos:**
- ~~`pendingApprovals` no tiene columna para "solicitado por `<agente>`" ni para inputs externos~~ — **cerrado 2026-08-04**, ver sección dedicada abajo (Jin_Core PR #21 + Jin_Web PR #4).
- No existe una tool/endpoint de "comentar código línea por línea" para el editor — se pidieron comentarios vía `/api/chat` con un formato de línea numerada parseado client-side (regex), documentado en el código como puente, no como feature con contrato propio.
- Íconos PWA son el favicon placeholder del template (`favicon.svg`), no el ícono 512px maskable que pide el diseño §09 — necesita un asset real, no generable en esta sesión.
- El widget "sesión actual" del panel de presupuesto (v1/v2 del diseño) no tiene equivalente backend limpio — ya resuelto en v3 reemplazándolo por "última hora", que sí usa datos reales de la Fase 6.1.1.

## Auditoría de cobertura: BLUEPRINT vs. roadmap (2026-08-05)

El owner preguntó si las fases pendientes cubrían todas las funciones que Jin debería tener. Se auditó `BLUEPRINT.md` completo (§1-15) contra el código real de los 6 repos y contra el roadmap de `STATUS.md`/`PROMPTS.md`. **Resultado: 11 funciones especificadas en el blueprint que ninguna fase 1-7 cubría.** Cada una verificada contra el código, no contra el texto de otro documento:

| Función | BLUEPRINT § | Estado real verificado | Resolución |
| --- | --- | --- | --- |
| Notion (integración completa) | §1.1, §7.4, §8.2, §9.1 | Cero código. Es un objetivo declarado del proyecto. | **Fase 9.1** |
| Comando `/audio` (transcribe → Notion) | §8.2 | No está entre los 10 comandos reales del bot | **Fase 9.1** |
| GitHub App + tools git | §7.3, §9.1 | Cero código en Core y Executor. `mergeAgentBranch` lanza 501 con el comentario "no existe hoy ninguna tool que le dé a un sub-agente la capacidad de producir una branch real" | **Fase 9.2** |
| RAG de corpus propio (pgvector) | §3.3, §3.3.1, §6.4 | Solo existe la mitad `sqlite-vec` (memoria, Fase 4.3). pgvector está en la imagen pero ningún módulo lo usa. | **Fase 9.3** |
| Alerta matutina 06:00 | §7.1 | El cron 00:00 (Shadowing) sí existe; el de 06:00 no, ni nada que arme "resumen de prioridades" | **Fase 9.4** |
| Infisical SDK en runtime | §11 | Manifest desplegado (Fase 1.1) pero **cero SDK en el código**: hoy todo es env plano, exactamente lo que §11 prohíbe | **Fase 8.1** (antes de 7.2) |
| Golden set prompt injection (~50) | §13.1 **obligatorio** | Cero. `injection-sanitizer.spec.ts` cubre la unidad, no un corpus adversarial | **Fase 8.2** (antes de 7.3) |
| Feature flags (ConfigMap + SIGHUP) | §12.3 | Cero código, cero ConfigMap | **Fase 9.5** |
| CLI: admin tasks + menús navegables | §8.3 | Fase 6.4 construyó 7 comandos de operación, ninguno de admin, sin menús de flecha | **Fase 9.6** |
| OpenTelemetry / tracing | §10.3 | Cero. Diferido en Fase 1 ("hasta que duela"), nunca retomado | **Diferido, con criterio explícito** (abajo) |
| PgBouncer | §3.3 | Cero. Diferido 2026-07-19 junto con Tempo | **Diferido, con criterio explícito** (abajo) |

**Ordenamiento no obvio, decidido en la auditoría:** `8.1` (Infisical SDK) va **antes** de `7.2` porque 7.2 escribe el runbook de "carga de secretos reales en Infisical" — si se escribe mientras el runtime lee env plano, documenta el mecanismo equivocado. `8.2` (golden set) va **antes** de `7.3` para que la auditoría de seguridad de 7.3 corra contra una base ya cubierta, en vez de descubrir agujeros que un test permanente ya debería estar vigilando.

**Diferidos con criterio de reactivación explícito** (para no repetir el patrón de "pospuesto y olvidado" que produjo esta auditoría):
- **OpenTelemetry/Tempo (§10.3):** entra cuando aparezca el primer bug de latencia o de causalidad entre servicios que los logs de Loki no alcancen a explicar. Antes de eso es instrumentación sin pregunta que responder.
- **PgBouncer (§3.3):** entra cuando Postgres muestre presión real de conexiones. Con 1 réplica de core y un solo usuario, hoy no hay caso.

**No son gaps** (desviaciones ya decididas y documentadas): BullMQ descartado (el diagrama de arquitectura de §2 quedó desactualizado, es solo el diagrama), Alertmanager reemplazado por Telegram directo (Fase 4.1), MCP servers y chaos tests ya cubiertos por 7.3, manifests de deploy ya cubiertos por 7.1.

## Ronda 2 de recomendaciones — auditoría cross-repo (2026-08-05)

El owner pidió revisar el proyecto entero buscando qué más falta que no estuviera documentado, tras cerrar los dos P0 de la Ronda 1. Se auditaron los 6 repos. **Resultado: 18 puntos nuevos (10-26 + 20.b)**, todos verificados contra el código con archivo:línea — el detalle completo está en `docs/RECOMENDACIONES.md`. Los que más importan:

- **P0 seguridad/integridad:** zip-slip en las claves de `files` de `startPreviewService` (+ esos pods sin `readOnlyRootFilesystem`); **tres notificaciones críticas que quedaron esperando "Fase 2.4"** — una fase cerrada hace semanas — así que el escalado de HITL a 12h, el abandono a 24h y la detección de audit chain corrupta solo escriben un log que nadie mira, pese a que el canal de Telegram proactivo ya funciona (lo usan las alertas de presupuesto); el lock del audit log por corrupción es en memoria y un redeploy lo resetea (gap que el propio repo ya reconocía **solo en comentarios de código**, nunca documentado hasta ahora); `audit_log` pierde `actor`/`external_inputs_summary` al resolver la aprobación; la contraseña de `jin login` viaja como argumento de shell; y solo 1 de los 3 tipos de backup se verifica restaurable.
- **P0 encontrado al final, con consecuencia ya materializada:** `Jin_CLI/source/api-types.ts` está desincronizado desde el PR #21 — le faltan `actor`/`externalInputsSummary`, y `TasksView` no los muestra. **Quien aprueba desde la terminal ve menos información que quien aprueba desde el dashboard.** Nadie se enteró porque el CI de `Jin_CLI` corre `pnpm run test || true` y `generate:api || true`.
- **P1:** ningún cliente maneja `disconnect` del WebSocket (la UI queda colgada en "pensando" para siempre); las mutaciones de `ApprovalCard` no tienen `catch`, así que una aprobación fallida se ve igual que una exitosa; `js-yaml` de producción con advisory alto en Core; `monaco-editor`→`dompurify` vulnerable en Web.
- **P2:** `Jin_Web` no tiene un solo test de ningún tipo; en `Jin_Core` faltan specs en piezas de seguridad (`chain-verification.service.ts`, `ws-token.ts`, los dos gateways WS, `env.schema.ts`).

**Corrección de un dato reportado antes:** `Jin_Infra` **sí** tiene CI, y es de los más completos (`kustomize build` + `kube-linter` + `shellcheck` + tests `bats`). El repo sin CI es `Jin_Docs`.

**Verificaciones que dieron limpio** (para no re-auditar): el secret del webhook de Telegram ya es requerido con validación estricta (el feedback de la Ronda 3 a Antigravity quedó bien resuelto); `Jin_Web` maneja el JWT correctamente (cookie `__Host-`, nunca en `localStorage` ni en logs); no hay secretos en texto plano en los manifests de `Jin_Infra` y todos los workloads declaran `resources.limits`; los tipos generados de `Jin_Web` están en sincronía con el contrato; y no hay `TODO`/`any` sin justificar en el código de negocio de Core ni Executor.

## Recomendación #1 — readiness/liveness probes reales (Jin_Core PR #22, 2026-08-04)

Primera de las 9 recomendaciones de `docs/RECOMENDACIONES.md` en implementarse (P0, va antes o junto con Fase 7.1). El único endpoint de salud era `GET /`, un string estático: apuntar ahí la readinessProbe que BLUEPRINT §12.2 declara obligatoria haría ficticio el "cero downtime" del rolling update `maxSurge:1/maxUnavailable:0`.

- `GET /health/live` no consulta dependencias a propósito (reiniciar el pod no arregla un Postgres caído); `GET /health/ready` sí: Postgres es dependencia dura (`error` + 503), Redis solo degrada (`degraded` + 200). La asimetría es coherente con el fail-open ya decidido en ADR 0007 #4 — sacar el pod de rotación por Redis contradiría esa decisión justo cuando más importa.
- Ambas `@Public()` (el kubelet no tiene JWT) y `@SkipThrottle()` (las probes salen de la IP del nodo; un 429 se lee como pod caído). Los 4 e2e nuevos verifican exactamente eso contra el pipeline HTTP real — un unit spec del controller no ve el guard global.
- `RedisThrottlerStorage.isReachable()` reusa la única conexión Redis del repo en vez de abrir una segunda para sondear. Su integration spec es el **primer consumidor real de `test/support/redis-testcontainer.ts`**, que estaba escrito desde antes sin ningún test que lo usara.
- Timeout propio de 2 s por chequeo: `pg.Pool` sin `connectionTimeoutMillis` reintenta hasta el timeout TCP del SO (minutos).

**Handoff a Fase 7.1 (Antigravity):** los manifests deben apuntar `livenessProbe` → `/health/live` y `readinessProbe` → `/health/ready`, **no** `/`.

**Verificación:** 347 unit (57 archivos) + 23 e2e (19 previos + 4 nuevos), `tsc --noEmit -p tsconfig.json` (incluye specs), `lint` y `build` limpios, contrato OpenAPI regenerado (+73 líneas). El integration spec de Redis no se pudo correr localmente (daemon de Docker apagado en la máquina) — verificado en el CI del PR, que sí corre `test:integration`.

## Poda y compresión del historial de chat (Jin_Core PR #23, 2026-08-04)

Recomendación #2 de `docs/RECOMENDACIONES.md` (P0) + requisito propio del owner (2026-08-04): no basta con podar, hace falta **comprimir los chats periódicamente para no agotar el contexto**. Implementada.

**Diseño (`src/agent/history-compaction.logic.ts` + `.service.ts`):** criterio híbrido — techo de tokens estimados (`config/agent.yaml: max_history_tokens`) + los últimos `preserve_last_turns` turnos **siempre** quedan verbatim (piso `.min(1)`, no solo UX — es la garantía de que el turno actual, con tags `<untrusted_content_{nonce}>` recién creados, nunca entra a una llamada de compresión el mismo turno en que se generó). El tramo viejo se resume (TaskProfile `history_compaction` nuevo, **sin `tools` declaradas** — garantía estructural de que la compresión no puede invocar ninguna tool real) y se consolida a `src/memory/` — **primer consumidor real de `MemoryService.consolidate()`** desde Fase 4.3 (handoff que llevaba semanas sin dueño). Ninguna de las dos llamadas puede romper el turno: fallos se loguean y degradan a "no se comprimió esta vez".

**El hallazgo que simplificó el diseño respecto a lo planteado inicialmente:** se investigó si hacía falta que `summarizeUntrustedSources(messages)` (`agent.service.ts`, PR #21 — puebla el "Influido por" del HITL) sobreviviera a la poda de `input.history`. **No hace falta.** Verificado: el único cliente real que manda `history` entre turnos (`Jin_CLI/ChatView.tsx`) solo persiste `{role, content: string}` con el `finalResponse` final de cada turno — nunca los bloques `tool_result` intermedios que llevan los tags `<untrusted_content_{nonce}>`. Esos tags mueren al final de cada `runTurn()`, nunca cruzan al cliente, nunca vuelven en `history` — podar `input.history` entre turnos no toca ni puede tocar la trazabilidad HITL, que se calcula en vivo cada turno sobre datos que la poda ni ve.

**El riesgo real de seguridad, más acotado que la hipótesis inicial:** el LLM que resume el tramo viejo, y `ConsolidationService.distill()` (dormido, sin caller real hasta este PR), leen texto de turnos anteriores que puede citar textualmente contenido externo que el turno original ya vio envuelto y con instrucción defensiva — pero ninguno de esos dos prompts la tenía. Se resolvió con `buildGenericUntrustedContentInstruction()` nueva (`src/security/injection-sanitizer.ts`, variante sin nonce de la instrucción que ya usaba `agent.service.ts`, para prompts que operan offline sobre texto ya cerrado), agregada a ambos — no envolviendo el resultado con `wrapUntrustedContent` (ese mecanismo es para tool output crudo del turno actual con nonce de sesión activo, no aplica semánticamente a un resumen de conversación pasada).

**Contrato:** `AgentTurnResult` gana `compactedHistory?` — como `/api/chat` es stateless (Fase 6.1, sin persistencia de sesión server-side), es la única forma de que la compresión reduzca algo real: el caller la **adopta** (reemplaza, no concatena) como su historial para el próximo turno. Campo aditivo/opcional, contrato OpenAPI regenerado.

**Fuera de alcance, documentado — no se toca en este PR:**
- **Handoff a Antigravity:** `Jin_CLI` necesita adoptar `compactedHistory` (reemplazar su array local por el valor devuelto cuando esté presente) para que la compresión tenga efecto real en el único cliente que hoy acumula historial sin cota. Sin este adopt, el fix queda construido pero sin consumidor.
- **Gap preexistente encontrado, no relacionado:** `Jin_Web/app/features/chat/useChat.ts` hoy **no manda `history` al backend en absoluto** (solo `{sessionId, objective}`) — cada turno de Jin_Web ya es de facto sin memoria conversacional, un bug distinto ("se olvida todo", no "crece sin cota") que este trabajo no resuelve.

**Verificación:** 364 unit (61 archivos, incluye caso adversarial: transcript con instrucción embebida no cambia el comportamiento del código, la defensa es de prompt) + 20 e2e, `tsc --noEmit -p tsconfig.json`, `lint` y `build` limpios, contrato OpenAPI regenerado (+111 líneas), `docs/MODEL_ROUTING.md` actualizado con el profile nuevo. `test:integration` no se pudo correr localmente (Docker apagado en la máquina) — verificado en el CI del PR.

## HITL: actor + inputs externos en pendingApprovals (Jin_Core PR #21 + Jin_Web PR #4, 2026-08-04)

Cierra el gap documentado en Fase 6.2+6.3: `pendingApprovals` no tenía columna para "solicitado por `<agente>`" ni para inputs externos — `audit_log` sí las tiene (`actor`/`external_inputs_summary`), pero solo se escriben ahí DESPUÉS de resolverse la aprobación. Sin esto, `GET /api/hitl/pending` no cumplía `AGENTS.md` §5.1 punto 3 en el momento en que más importa: antes de que el owner decida, no solo en el rastro posterior.

- **Migración 0006** (`pending_approvals.actor`/`external_inputs_summary`, mismos nombres que `audit_log`). Hallazgo real en el camino: `drizzle-kit generate` producía una migración que recreaba 5 tablas ya existentes — el snapshot chain de drizzle-kit (`drizzle/meta/*.json`) nunca tuvo entradas para las migraciones 0003-0005 (se escribieron a mano, sin snapshot, precedente ya establecido en el repo). Se descartó el auto-generado y se escribió 0006 a mano siguiendo el mismo patrón (`CREATE ... IF NOT EXISTS` / `ALTER ... ADD COLUMN IF NOT EXISTS`).
- **`summarizeUntrustedSources()`** (nuevo, `src/security/injection-sanitizer.ts`): contraparte de lectura de `wrapUntrustedContent` (Fase 5.1/ADR 0004). En `agent.service.ts::runTurn`, el array `messages` nunca se poda — al clasificar una tool `confirm`/`dual-confirm` en la iteración N, ya contiene los `tool_result` envueltos (`<untrusted_content_{nonce} source="...">`) de las tools `auto`/`notify` de las iteraciones 1..N-1 del mismo turno. La función extrae esos `source` (regex sobre el tag) y cuenta ocurrencias — la traza real de qué pudo influir en la decisión del LLM, no un resumen inventado.
- `agent.service.ts` pasa `actor`(=`actorLabel`)/`externalInputsSummary` reales. `orchestrator.service.ts` (conflictos entre sub-agentes) pasa solo `actor:'orchestrator'` — ese path no corre un turno de LLM (compara tickets ya completados), así que no hay `messages` del que extraer nada; forzar un valor ahí sería inventar dato, documentado así en el código.
- `Jin_Web/ApprovalCard.tsx` muestra ambos campos ("Solicitado por: ...", "Influido por: ...") cuando no son null.

**Verificación real, no solo tipos:** 337 unit + 68 integración (Postgres real vía testcontainers, incluye el path de orchestrator) + 19 e2e (contrato serializado por el pipeline real) en Jin_Core; `typecheck`/`lint`/`build` limpios en Jin_Web. Smoke manual con el stack local (`docker-compose.dev.yaml`): fila real insertada en Postgres, Playwright confirma que la card del dashboard muestra "Solicitado por: web-chat" / "Influido por: readEmails (2), listCalendarEvents (1)" con los valores reales, no placeholders.

## Local dev tooling — Postgres/Redis (Jin_Infra PR #7 + Jin_Core PR #20, 2026-08-04)

Cierra el "no verificado" que quedó anotado en Fase 6.2+6.3 (línea arriba): el owner pidió alojar Postgres/Redis local para probar el dashboard hoy mismo, no una decisión de hosting de producción.

- **`Jin_Infra/docker-compose.dev.yaml`** (nuevo, PR #7) — no existía pese a estar referenciado desde antes en `Jin_Core/CLAUDE.md` y `Jin_Docs/CLAUDE.md` §4. Mismas imágenes que producción (`pgvector/pgvector:0.8.5-pg16`, `redis:7.4.9-alpine`), credenciales fijas de desarrollo, documentado en el propio archivo como nunca-producción. Nota: es una edición fuera del área habitual de Claude Code en este repo (`scripts/backup/**` según `Jin_Infra/CLAUDE.md`) — mismo criterio ya usado en Fase 5.5 para excepciones puntuales pedidas por el owner.
- **`Jin_Core/.env.example`** (PR #20) — estaba desactualizado desde antes de Fase 6.1 (solo 5 de las 16 vars de `env.schema.ts`). Ahora completo, con placeholders dev-safe y puntero al compose de arriba.
- **Gap real encontrado:** `src/db/migrate.ts` (y el resto de scripts standalone: `generate-contract.ts`, `hash-password.ts`) nunca cargan `.env` — a diferencia del `ConfigModule.forRoot()` de NestJS, que sí lo hace automático para el server. `pnpm run db:migrate` fallaba con todas las vars `undefined` hasta exportar `.env` al shell manualmente (`set -a; source .env; set +a`). No se agregó `dotenv`/`dotenv-cli` como dependencia nueva (requeriría preguntar al owner, CLAUDE.md §7) — queda documentado acá como el paso manual necesario hasta que se decida si vale la pena la dependencia.
- **`start:dev` de Jin_Core necesitó `NODE_OPTIONS=--max-old-space-size=4096`** — el heap default de V8 (~2GB) no alcanza para `tsc --watch` de este tamaño de proyecto en esta máquina (8GB RAM total, memoria compartida con Antigravity IDE/VS Code). Sin este flag el proceso muere con `FATAL ERROR: Reached heap limit`.

**Verificación real, cerrando el gap de Fase 6.2+6.3:** con ambos containers `healthy`, migraciones aplicadas (10 tablas), `Jin_Core` (`start:dev`, puerto 3000) y `Jin_Web` (`pnpm dev`, puerto 5173) corriendo — Playwright headless: login con `OWNER_PASSWORD_HASH` real, las 9 rutas (overview/hitl/chat/budget/audit/memory/board/editor/apps) cargan con datos reales (presupuesto `$0.00/$10.00 · límite 5,000,000 tokens`, WS "EN VIVO" conectado), sin pantallas negras ni errores de consola reales. Contraseña de desarrollo: `jin-dev-2026` (hash en `.env`, gitignored).

## Fase 6.1.1 — resumen técnico (Jin_Core PR #18, mergeado 2026-08-03)

Encontrado revisando el diseño v2 del dashboard contra el código real, antes de empezar la Fase 6.2 (Jin_Web) — no era un requisito de PROMPTS.md, salió de verificar que la API soportara lo que el diseño ya asumía.

**1. `GET /api/budget` extendido con datos reales.** El endpoint de Fase 6.1 solo devolvía `{dailyUsageRatio, killSwitchActive}` — el panel de presupuesto diseñado necesita montos ($, tokens) y el detalle de por qué se activó el kill switch. `BudgetService`/`KillSwitchService` ya calculaban estos datos internamente (`checkRunaway()` los compara cada 5 min); solo faltaba exponerlos: `BudgetService.getDailyUsage()`/`getLimits()` y `KillSwitchService.getStatus()` nuevos (públicos), reusando la consulta de horas en un solo lugar en vez de duplicarla.

**2. El contrato OpenAPI de toda la Fase 6.1 estaba vacío — hallazgo mayor.** Al regenerar el contrato para verificar (1), salió byte-idéntico al anterior. Investigando: **0 de los 10 endpoints propios de Fase 6.1** (auth/chat/hitl/audit/budget/memory) tenía request/response documentado — interfaces TS planas sin decoradores no son introspectables por `@nestjs/swagger`. Esto habría hecho que `pnpm generate:api` en Jin_Web generara tipos vacíos para todo, no solo para budget — contradice la regla de oro #11 antes de escribir la primera línea del dashboard.

**Decisión del owner: adoptar `nestjs-zod`**, no DTOs a mano con `@ApiProperty` (ver "Decisiones del owner"). Reemplaza el `ZodValidationPipe` casero (borrado): cada schema Zod que ya validaba bodies pasa por `createZodDto()` y sirve para validación + tipo TS + doc OpenAPI desde una sola definición. Wiring global en `app.module.ts` (`ZodValidationPipe`/`ZodSerializerInterceptor` como `APP_PIPE`/`APP_INTERCEPTOR`) y `main.ts`/`generate-contract.ts` (`cleanupOpenApiDoc()`).

**Dos problemas técnicos reales encontrados migrando, no anticipados:**
- **Bug real preexistente:** `AuditLogRow.id` es `bigint` (`bigserial({mode:'bigint'})`) — `JSON.stringify` no serializa `bigint` nativamente, así que `GET /api/audit` habría reventado con cualquier fila real en producción. Nunca se detectó porque ningún test serializaba una fila real por HTTP (los specs solo verificaban delegación con arrays vacíos). Corregido: `id` se convierte a `string` en el controller, mismo criterio que ya usaba `nextCursor`. Cubierto con un e2e nuevo que usa un bigint real de punta a punta.
- **`z.date()` no es representable en JSON Schema en Zod v4** (`hitl`/`audit` tienen columnas `timestamp` que Drizzle deserializa como `Date` real, no string). Resuelto con codecs (`src/common/dto/date-codec.ts`: `dateCodec`/`nullableDateCodec`) — el lado documentado es un string ISO 8601, `.encode()` es lo que `nestjs-zod` usa para responses cuando el DTO se crea con `{ codec: true }`, convirtiendo el `Date` real al responder.

**Verificación (2026-08-03):** `tsc --noEmit` sin errores, `pnpm lint` verde, 325 tests unitarios (53 archivos, +1 sobre Fase 6.1) + 65 de integración (Postgres real, +4) en verde, `pnpm build` limpio. **14 tests e2e nuevos/actualizados** en `test/app.e2e-spec.ts` — a diferencia de los unit specs (que mockean el controller directo, sin pasar por el pipeline HTTP real), estos bootstrapean `AppModule` completo vía `supertest` con el guard JWT real y `ZodSerializerInterceptor` real: sin esto, el bug de `bigint` y el problema de `z.date()` no se habrían detectado — ambos solo se manifiestan en el camino real request→pipe→handler→interceptor→`JSON.stringify`, no en un mock. `gh pr checks` real confirmado en verde (`build` 2m3s, GitGuardian pass) antes de mergear.

Ver `docs/adr/` — no ameritó ADR nuevo (extensión de Fase 6.1, no una decisión arquitectónica nueva); documentado aquí y en el propio PR #18.

**Fuera de alcance, documentado:** no se auditó el kill switch (`KillSwitchService.checkRunaway()` detecta y persiste el estado, pero no escribe una fila en `audit_log` al activarse — solo se nota vía polling de `isActive()`/`getStatus()`). El owner priorizó cerrar el gap de budget/contrato primero; auditar la activación del kill switch queda como mejora futura si se decide que vale la pena.

## Fase 6.1 — resumen técnico (Jin_Core PR #17, mergeado 2026-08-02)

Auth JWT single-user + API REST/WebSocket (BLUEPRINT §4.1/§5.2), área de Claude Code. Cierra la dependencia que bloqueaba Fase 6.2 (Web Dashboard) y 6.4 (CLI): hasta ahora todo el sistema solo se veía por Telegram.

**`src/auth/`:** `AuthService` (Argon2id vía `argon2.verify` contra `OWNER_PASSWORD_HASH`, firma JWT con `@nestjs/jwt`), `JwtAuthGuard` registrado como `APP_GUARD` global (protege todo por default), `@Public()` para el allowlist explícito (`/api/auth/login`, `/metrics`, `/telegram/webhook`, liveness check en `AppController`). Entrega dual: cookie `__Host-jin_session` (httpOnly/secure/sameSite=strict, sin `Domain`) para el Web, mismo token en el body de login para la CLI. Sin blocklist de logout en v1 (límite conocido, aceptado — ver ADR 0007 punto 2); logout solo borra la cookie cliente, expiry de 7 días acota el riesgo.

**`JinErrorFilter`** (`src/common/filters/`), `APP_FILTER` global: traduce `JinError.httpStatus` a la respuesta HTTP real — prerequisito real encontrado durante la investigación (ningún controller existente lo necesitaba porque Telegram nunca propagaba `JinError` directo a una respuesta HTTP).

**Rate limiting** (`src/rate-limit/`): primera vez que Jin_Core habla con Redis (ya desplegado en Jin_Infra, sin infra nueva). `@nestjs/throttler` + storage propio sobre `ioredis` (ventana fija, `INCR`+`EXPIRE` atómico vía Lua `EVAL` — decisión explícita de no implementar sliding window real, ver ADR 0007 punto 4). 60/min global, 5/15min en `/api/auth/login`.

**Controllers de dominio** (dentro de cada módulo existente, no un módulo "api" genérico — AGENTS.md 4.2): `hitl.controller.ts` (`GET /api/hitl/pending`, `POST /api/hitl/:requestId/approve|reject`, delega en `ApprovalExecutionService` ya existente), `audit.controller.ts` (`GET /api/audit?cursor&limit`, nuevo `AuditService.listRecent()`), `budget.controller.ts` (`GET /api/budget`, `POST /api/budget/unpause`), `memory.controller.ts` (`POST /api/memory/recall`). `DualConfirmService` ganó `listPending()` + emit de `EventEmitter2` en `createPendingApproval()`.

**`src/chat/`:** `POST /api/chat` + gateway WS delgado sobre `AgentService` ya existente, sin persistencia de sesión server-side (decisión deliberada — el caller gestiona su propio historial, a diferencia de Telegram).

**`src/realtime/`:** gateway híbrido, no todo push ni todo polling. `pending-approval:new` es genuinamente event-driven (`@nestjs/event-emitter`, único emit point real en `DualConfirmService`). `budget:alert`/`kill-switch:activated` replican el mismo sondeo por `@Cron` que ya usa `TelegramBotService.checkBudgetAlerts()` (mismo dedup por umbral/día) — no se extrajo esa lógica a un servicio compartido todavía (AGENTS.md 1.1: esperar 3 consumidores antes de abstraer; con Telegram + este gateway son 2).

**`main.ts`:** Swagger UI movida de `/api` a `/docs` (libera el path que BLUEPRINT 5.2 reserva para la API de negocio real), `.addBearerAuth()`, `cookie-parser` registrado.

**Verificación (2026-08-01/02):** `tsc --noEmit` sin errores, `pnpm lint` verde, 324 tests unitarios (53 archivos) + 61 de integración (Postgres + Redis reales vía testcontainers) en verde, `pnpm build` limpio, `contracts/openapi.json` regenerado (188 líneas, puramente aditivo: `/api/auth/*`, `/api/chat`, `/api/hitl/*`, `/api/audit`, `/api/budget/*`, `/api/memory/recall`). `gh pr checks` real confirmado en verde (`build` 2m18s, GitGuardian pass) antes de mergear.

**Nota de infraestructura local:** `pnpm lint`/`pnpm run generate:contract` necesitaron `NODE_OPTIONS=--max-old-space-size=8192` en la máquina de desarrollo local (heap default insuficiente para el type-aware lint de todo el repo) — no reprodujo en el runner de GitHub Actions (CI verde sin el flag), documentado por si vuelve a aparecer.

Ver ADR 0007 (`docs/adr/0007-auth-jwt-api.md`) para el detalle de cada decisión y las alternativas descartadas.

**Fuera de alcance, documentado:** streaming de tokens en el chat WS no existe (`ModelProvider` no lo soporta hoy — la respuesta llega completa cuando el turno termina, no token por token, límite real de PROMPTS.md ya anotado en el plan).

## Fase 5.5 — resumen técnico (Jin_Executor PR #5 + Jin_Core PR #16 + Jin_Infra PR #6, mergeados 2026-08-01)

Pods de servicio de larga vida (BLUEPRINT §4, requisito del owner 2026-07-25, ADR 0006): los agentes pueden levantar procesos persistentes (`npm run dev`, un backend de prueba) con puerto expuesto bajo `https://<slug>.jinserver.com` — nunca `jeanfranck.com` (regla de oro #10).

**Decisión de diseño confirmada con el owner en esta sesión:** `PROMPTS.md` §5.5 sugiere "código clonado de su branch", pero ninguna tool existente le da a un agente acceso a un repo git real (mismo gap que ADR 0005 dejó documentado para `mergeAgentBranch`). `startPreviewService` recibe `files: Record<string,string>` inline; un init container los extrae de un tar.gz (armado a mano — sin librería nueva, verificado por round-trip contra el `tar` real del sistema) sin ConfigMap ni riesgo de inyección de shell (el alfabeto base64 excluye por construcción cualquier metacaracter).

**`Jin_Executor` (PR #5):** nuevo `src/preview-service/` — `PreviewServiceLifecycleService` (`start`/`stop`/`list`) opuesto a `PodLifecycleService`: nunca destruye en un `finally`, no espera un estado terminal. Kubernetes ES el store de estado (TTL como annotation `jin.io/expires-at`, slug como `jin.io/slug` — sin Postgres nuevo en este repo). Reaper (`@Cron('*/5 * * * *')`, dependencia nueva `@nestjs/schedule`) destruye servicios vencidos. Primer uso en el repo de un CRD de terceros (`IngressRoute` de Traefik, grupo `traefik.io`/`v1alpha1`) vía `CustomObjectsApi`. `NetworkPolicy` de ingreso nueva scoped por pod (agents-sandbox deniega `Ingress` por default — sin ella Traefik nunca alcanzaría el pod). PVC compartido `pnpm-store` (RWO, viable porque el clúster es de un único nodo). `ExecutorToolDefinition` pasa a discriminated union (`RunToCompletionToolDefinition` vs. `ServiceToolDefinition`) para narrowing real sin non-null assertions.

**`Jin_Core` (PR #16):** 3 tools nuevas en `registry.ts` (`startPreviewService` confirm, `stopPreviewService` notify, `listPreviewServices` auto) + `ExecutorClientService`/`ExecutorClientModule` extendidos hablando con el nuevo `POST/DELETE/GET /services` del Executor — mismo patrón exacto que `runCode` (Fase 5.2).

**`Jin_Infra` (PR #6):** RBAC del `ServiceAccount` `executor` gana `services`/`ingressroutes.traefik.io` (create/get/list/delete) SOLO en `agents-sandbox`, sin tocar `ResourceQuota`/`LimitRange` (restricción explícita de PROMPTS.md). `Certificate` nuevo para `*.jinserver.com` en `agents-sandbox` (el Secret TLS de `jin` no es visible ahí). Script de bootstrap nuevo (`07-sysctl-inotify.sh`) para `fs.inotify.max_user_watches` — sysctl de nodo, no seteable por pod, sin el cual el hot reload falla en silencio. **Cambio de alcance:** toca `k8s/base/executor/**`/`scripts/bootstrap/**`, normalmente área de Antigravity — acotado a una frontera de seguridad (RBAC), mismo precedente que backups en Fase 1.2, explícitamente anticipado por `PROMPTS.md` §5.5.

**Tests de integración con K3s real:** CRD de `IngressRoute` registrado a mano en el test (`@testcontainers/k3s` corre con `--disable=traefik` por defecto — sin controller de Traefik reconciliándolo, pero el apiserver real valida y persiste el manifest de todos modos), pod+Service responden HTTP verificado desde un probe en `kube-system` (único namespace que la NetworkPolicy de ingreso permite), reaper destruye pod+Service+NetworkPolicy vencidos.

**CI real, dos fallas intermitentes encontradas y resueltas antes de mergear (`gh pr checks` verificado, no solo local):**
1. El probe de "camino feliz" fallaba en CI (no en local) con `TypeError` al hacer `fetch` al `Service` recién creado — CoreDNS tarda un instante en propagar el registro bajo la carga de un runner de CI. Corregido con reintentos (5x) que distinguen esa latencia de un bloqueo real de la NetworkPolicy.
2. Una segunda corrida falló en el test de aislamiento de red **preexistente de Fase 2.3** (`pod-lifecycle.service.integration.spec.ts`, no tocado en este PR) — misma clase de flakiness ya documentada en ADR 0003 (containerd anidado). Reintentado el job de CI (no el test — es seguridad crítica, no se toca sin necesidad real) y pasó, confirmando que era transitorio.

**Fuera de alcance, documentado (ADR 0006):** clonar código desde una branch git real (texto literal de PROMPTS.md) queda pendiente de que exista una tool de edición de código — mismo patrón que `mergeAgentBranch` (ADR 0005 punto 9).

## Fase 5.4 — resumen técnico (Jin_Core PR #15, mergeado 2026-08-01)

Orquestación multi-agente (BLUEPRINT §6/§9, requisito del owner 2026-07-25, ADR 0005). Extiende el agent loop de Fase 5.1 sin reescribirlo: un sub-agente es una instancia más de `AgentService.runTurn()`, no una clase nueva.

**`OrchestratorService.runObjective()`:** crea un run en el ledger, descompone el objetivo en tickets vía LLM (`TicketDecompositionService`, TaskProfile `reasoning_heavy`), corre lotes de tickets listos (dependencias resueltas) con `Promise.all` en chunks de `max_concurrent_sub_agents` (config, default 3), y cierra con una pasada de reconciliación (`ReconciliationService`, mismo TaskProfile) que redacta la respuesta final y detecta contradicciones entre sub-agentes.

**Ledger persistido:** 3 tablas nuevas (`agent_orchestration_runs`/`agent_tickets`/`agent_ticket_comments`, migración `0004_agent_ledger`) — sobrevive restarts, mismo criterio que `pending_approvals`. `LedgerRepository` es el único punto de acceso.

**Escalera de decisión:** conflicto de bajo riesgo → el orquestador lo resuelve en modo-auto (comentario `resolution` con el porqué); conflicto material → tool sintética `resolveAgentConflict` (`hitlLevel: 'confirm'`) crea una pending approval real, reusando el HITL/Telegram existente sin canal nuevo.

**Presupuesto:** cada sub-agente corre con `sessionId = {runId}:{ticketId}` (techo de sesión propio, sin poder evadir los topes globales diario/horario/kill-switch). `BudgetService.getSessionUsage()` (nuevo, público) permite sumar el consumo real del turno completo.

**Board causal:** un sub-agente ve, al arrancar, el resultado de tickets ya `done` de lotes anteriores (inyectado como par sintético `user`/`assistant`, mismo mecanismo que la memoria de Fase 5.3) — nunca de tickets corriendo en el mismo lote en paralelo. Esa limitación es deliberada: las contradicciones entre pares del mismo lote se detectan recién en la reconciliación final, no a mitad de vuelo.

**Cambios retrocompatibles a `src/agent/agent.service.ts` (Fase 5.1):** `AgentTurnInput` gana `allowedTools?`/`actorLabel?` opcionales — ausentes, el comportamiento es idéntico al de antes de esta fase (todos los tests de Fase 5.1/5.3 preexistentes siguen pasando sin tocarlos).

**Bug real encontrado en el smoke test de `AppModule` completo** (no solo `tsc`/`test`/`test:integration`, que no lo detectaron): `AGENT_CONFIG` no estaba exportado desde `AgentModule` — `OrchestratorModule` no podía resolverlo, hubiera reventado recién al bootstrapear en real. Corregido exportando el token además de `AgentService`.

**Fuera de alcance, documentado (ADR 0005 punto 9):** `mergeAgentBranch` está declarada (`hitlLevel: 'confirm'`, guardrail exigido por PROMPTS.md) pero su executor lanza `AgentBranchMergeNotImplementedError` (501) — hoy ninguna tool le da a un sub-agente la capacidad de producir una branch de código real. Se activa cuando exista una tool de edición de código.

**Tests:** 284 unitarios + 59 de integración (Postgres real vía testcontainers) en el repo tras el merge, cubriendo los 6 escenarios exigidos por PROMPTS.md §5.4 (orden por dependencias, paralelismo real, conflicto material dispara pending approval, un `confirm` no bloquea tareas independientes, kill switch corta todo el run, presupuesto agregado = suma de sub-agentes) más el camino de conflicto de bajo riesgo y el de un ticket que falla.

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

- 2026-08-04: [Jin_Web] [PR #3](https://github.com/JFrnck/Jin_Web/pull/3) mergeado — Fase 6.2+6.3 completa: dashboard con las 10 pantallas del diseño v3 aprobado (no un MVP recortado, a pedido explícito del owner), reemplazando el template de Vite. React Router v8 (SPA, `ssr:false`), TanStack Query, Socket.IO, tipos generados desde el contrato real de Jin_Core. Paleta de riesgo HITL validada como ramp ordinal con el skill de dataviz. Bug real encontrado verificando en navegador real (Playwright headless): `openapi-fetch` resolvía `data:undefined` sin `error` ante fallos de red, dejando `/hitl` en pantalla negra — corregido con `unwrap()` centralizado + manejo de `isError` explícito en 5 pantallas más. `pnpm typecheck`/lint/build limpios, CI real verde (19s). Gaps documentados sin rellenar con datos falsos: `pendingApprovals` sin actor/inputs-externos, editor sin tool dedicada de comentarios (puente vía `/api/chat`), íconos PWA placeholder. Pendiente real: probar contra Jin_Core corriendo local con datos genuinos. Ver sección dedicada arriba.
- 2026-08-03: [Jin_Core] [PR #19](https://github.com/JFrnck/Jin_Core/pull/19) mergeado — Fase 6.1.2: endpoints REST para el board de orquestación (`GET /api/orchestrator/runs`, `GET /api/orchestrator/runs/:runId`) y preview services (`GET`/`DELETE /api/preview-services`), sin lógica nueva (delegan en `LedgerRepository`/`ExecutorClientService` ya existentes). 330 unitarios + 68 integración + 19 e2e en verde. Ver sección dedicada arriba.
- 2026-08-03: [Jin_Core] [PR #18](https://github.com/JFrnck/Jin_Core/pull/18) mergeado — Fase 6.1.1: `GET /api/budget` extendido con montos reales ($, tokens, detalle de kill switch) y contrato OpenAPI de Fase 6.1 completo (0 de 10 endpoints documentados → 10 de 10, vía `nestjs-zod`, decisión del owner). Bug real encontrado y corregido: `AuditLogRow.id` (bigint) no serializable por JSON, `GET /api/audit` habría reventado con una fila real. 325 unitarios + 65 integración + 14 e2e (pipeline HTTP real: guard + interceptor, no solo mocks) en verde, `gh pr checks` real confirmado antes de mergear. Encontrado revisando el diseño v2 del dashboard (aprobado, sin pendientes) contra el código real. Ver sección dedicada arriba.
- 2026-08-02: [Jin_Core] [PR #17](https://github.com/JFrnck/Jin_Core/pull/17) mergeado — Fase 6.1 completa: Auth JWT single-user (Argon2id + cookie `__Host-`/Bearer) + API REST/WebSocket (`hitl`/`audit`/`budget`/`memory`/`chat` controllers, gateway realtime híbrido, rate limiting con Redis real). `JinErrorFilter` global cierra un gap real preexistente. 324 tests unitarios + 61 integración (Postgres+Redis reales) en verde, `contracts/openapi.json` regenerado, `gh pr checks` real confirmado (`build` 2m18s) antes de mergear. ADR 0007 escrito. Desbloquea Fase 6.2 (Web Dashboard) y 6.4 (CLI), ambas de Antigravity. Ver sección dedicada arriba.
- 2026-08-01: [Jin_Core] [PR #14](https://github.com/JFrnck/Jin_Core/pull/14) mergeado (Antigravity) — Fase 5.3 completa: sesiones reales de Telegram con Agent Loop + Memoria Extendida, persistencia Postgres del transcript, ventana deslizante (20 mensajes) para el modelo vs. consolidación con transcript completo al cerrar sesión, comando `/memory`. **Revisado y mergeado por Claude Code**: checkout real de la rama, merge contra `main` actual (Fases 5.4/5.5 ya mergeadas), encontró y corrigió una colisión real de migración (`0004_telegram_sessions` → `0005_telegram_sessions`, `0004` ya tomado por `agent_ledger`), un merge conflict real en `schema.ts`, y un estilo menor (`created!` → chequeo explícito). Verificado tras el fix con `tsc`/`lint`/298 unitarios/59 integración (Postgres real)/smoke test de `AppModule` completo — `gh pr checks` real confirmado antes de mergear, no solo local.
- 2026-08-01: [Jin_Executor] [PR #5](https://github.com/JFrnck/Jin_Executor/pull/5) + [Jin_Core] [PR #16](https://github.com/JFrnck/Jin_Core/pull/16) + [Jin_Infra] [PR #6](https://github.com/JFrnck/Jin_Infra/pull/6) mergeados — Fase 5.5 completa: pods de servicio de larga vida (`npm run dev`, etc.) expuestos bajo `https://<slug>.jinserver.com` con TTL obligatorio y reaper automático. Código servido vía archivos inline (tar.gz armado a mano, sin dependencia nueva) — decisión confirmada con el owner en esta sesión, ninguna tool le da hoy a un agente acceso a git real. Primer uso de un CRD de terceros (`IngressRoute` de Traefik) en el repo. RBAC extendido en Jin_Infra (cambio acotado en área normalmente de Antigravity, mismo precedente que backups Fase 1.2). ADR 0006 escrito. CI verificado con `gh pr checks` real en los 3 repos — dos flakiness intermitentes de CI encontradas y resueltas (retry de DNS en el test nuevo; re-run de un test de aislamiento preexistente de Fase 2.3, sin tocarlo). Ver sección dedicada arriba.
- 2026-08-01: [Jin_Core] [PR #15](https://github.com/JFrnck/Jin_Core/pull/15) mergeado — Fase 5.4 completa: orquestación multi-agente (`OrchestratorService`, task ledger persistido en Postgres, escalera de decisión bajo-riesgo/material vía HITL existente, kill switch corta todo el run, presupuesto agregado vía `BudgetService.getSessionUsage()` nuevo). ADR 0005 escrito. 284 unitarios + 59 integración (Postgres real) en verde, CI verificado con `gh pr checks` real. Bug de wiring (`AGENT_CONFIG` no exportado) encontrado y corregido en un smoke test de `AppModule` completo antes de abrir el PR. Ver sección dedicada arriba. **Pendiente:** PR #14 (Antigravity, Fase 5.3) debe renumerar su migración `0004_telegram_sessions` a `0005_*` antes de mergear (colisión de numeración, ver nota en "En progreso").
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
