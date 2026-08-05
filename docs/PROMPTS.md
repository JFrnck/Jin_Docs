# PROMPTS.md — Prompts iniciales por fase

> Prompts listos para copiar/pegar en **Claude Code** o **Antigravity** al inicio de cada fase o sesión.
> Cada prompt es autónomo: menciona los docs a leer, define el alcance, declara criterios de éxito y self-checks.
> El LLM debe **leer los docs referenciados antes de escribir código**. No lo asumas, dilo explícitamente en el prompt.
> Los docs viven en el repo `Jin_Docs` (carpeta `Jin/Jin_Docs/`). Los prompts indican en qué **repo** se trabaja.

---

## Cómo usar este documento

1. Antes de cada fase, copia el prompt correspondiente y pégalo como primer mensaje en la sesión del IDE.
2. Espera a que el LLM lea los docs y proponga un plan.
3. Revisa el plan. Si hay dudas, pídelo por escrito.
4. Solo entonces autoriza a implementar.

Formato de los prompts:

- **`[CLAUDE CODE]`** — pegar en Claude Code.
- **`[ANTIGRAVITY]`** — pegar en Antigravity IDE.
- Cada uno referencia los docs específicos que debe leer y el repo donde se trabaja.

---

## Fase 1 — Base de infraestructura

### 1.1 [ANTIGRAVITY] — Provisionar OCI + K3s + Traefik (repo: Jin_Infra)

```
Vas a arrancar la infraestructura base de Jin. Trabajas en el repo `Jin_Infra`.

ANTES DE ESCRIBIR CÓDIGO:
1. Lee `../Jin_Docs/docs/BLUEPRINT.md` sección 3 (Infraestructura y persistencia) y sección 5 (Dominios y exposición).
2. Lee `../Jin_Docs/AGENTS.md` completo.
3. Lee `../Jin_Docs/docs/WORKFLOW.md` sección 2 (mapa de ownership) para confirmar que este repo es tuyo.

TU TAREA:
Crear el setup inicial de infra que incluya:
- Manifests Kustomize para K3s: `k8s/base/` (Traefik, cert-manager, cloudflared, Postgres, Redis, Infisical).
- Overlays: `k8s/overlays/production/` con configs finales.
- Scripts en `scripts/bootstrap/` para: instalar K3s en la VM, configurar Cloudflare Tunnel, aplicar los manifests iniciales, y pre-pull de la imagen Deno en el nodo (no hay warm pool: los pods se crean bajo demanda y la imagen debe estar cacheada).
- README en la raíz del repo con el runbook paso a paso desde una VM Ubuntu limpia.

CRITERIOS DE ÉXITO:
- Todos los manifests pasan `kubectl apply --dry-run=server`.
- El README permite a un humano bootstrappear la VM desde cero en <2 h.
- Ningún secreto en los YAML: todos referencian Infisical.
- Postgres StatefulSet con volume 50 GB y extensión `vector` instalada en el init.
- cert-manager con ClusterIssuer Let's Encrypt configurado para challenge DNS-01, con wildcards para `*.jeanfranck.com` y `*.jinserver.com`.
- Cada Deployment tiene readiness/liveness probes y resources declarados.
- Actualiza `../Jin_Docs/STATUS.md` al empezar y al terminar.

RESTRICCIONES:
- Prohibido introducir dependencias no aprobadas.
- Prohibido usar `latest` como image tag: siempre versión pinneada.
- Prohibido `hostNetwork: true` en cualquier pod.

ANTES DE IMPLEMENTAR:
Propón el plan detallado (archivos que crearás, orden, decisiones no triviales). Espera mi aprobación.
```

### 1.2 [CLAUDE CODE] — Configurar backups y verificación (repo: Jin_Infra)

```
Vas a implementar el sistema de backups y su verificación. Trabajas en el repo `Jin_Infra` (excepción al ownership: los scripts de backup son críticos y los lidera Claude Code; coordina en STATUS.md).

ANTES DE ESCRIBIR CÓDIGO:
1. Lee `../Jin_Docs/docs/BLUEPRINT.md` secciones 3.3.1 (sqlite-vec) y 3.5 (Backups).
2. Lee `../Jin_Docs/AGENTS.md` completo, énfasis en sección 5 (Seguridad) y sección 6 (Testing).
3. Lee `../Jin_Docs/docs/WORKFLOW.md` para confirmar la coordinación.

TU TAREA:
Crear scripts y CronJobs para:
- `scripts/backup/dump-postgres.sh` — pg_dump custom format, cifrado con `age`, upload a R2 (bucket `jin-backups`).
- `scripts/backup/dump-redis.sh` — SAVE + copia RDB, cifrado, upload.
- `scripts/backup/dump-memory.sh` — `sqlite3 memory.db ".backup"`, cifrado, upload (memoria extendida sqlite-vec).
- `scripts/backup/verify-restore.sh` — el día 1 de cada mes: levanta contenedor Postgres temporal, aplica el dump más reciente, valida integridad con checks SQL, borra el contenedor, envía resultado a Telegram (stub por ahora).
- CronJobs de Kubernetes en `k8s/base/backup/`.
- Tests que validen que:
  - El cifrado age funciona correctamente y no se puede descifrar sin la master key.
  - La rotación de retención (7 diarios, 4 semanales, 3 mensuales) funciona.
  - El restore test detecta corrupción intencional.

CRITERIOS DE ÉXITO:
- El backup completo se ejecuta en <10 min.
- El upload a R2 tolera fallos de red (retry con backoff).
- El test de restore corre en CI de este repo en cada PR que toca este código.
- Cobertura de tests >90% en los scripts críticos.

RESTRICCIONES:
- La master key de age nunca aparece en logs.
- Los dumps nunca se escriben a disco sin cifrar. Streaming pipe: `pg_dump | age | rclone`.

ANTES DE IMPLEMENTAR:
Propón el plan. Especifica cómo vas a probar el restore end-to-end sin datos reales.
```

---

## Fase 2 — Núcleo del orquestador

### 2.1 [ANTIGRAVITY] — Scaffolding de los repos (repos: Jin_Core, Jin_Executor, Jin_Web, Jin_CLI)

```
Vas a inicializar los repositorios de Jin. NO es un monorepo: cada repo es autónomo, con su propio CI, lockfile y `.nvmrc`.

ANTES DE ESCRIBIR CÓDIGO:
1. Lee `../Jin_Docs/docs/BLUEPRINT.md` sección 4 completa (incluida 4.6, contratos entre repos).
2. Lee `../Jin_Docs/AGENTS.md` sección 4 (Arquitectura de código).
3. Lee `../Jin_Docs/docs/WORKFLOW.md` sección 2 (ownership por repo).

TU TAREA:
Para `Jin_Core`:
- NestJS con Zod validation, pino logger, auth JWT stub, health endpoint.
- `@nestjs/swagger` configurado + script `generate:contract` que emite `contracts/openapi.json`.
- Estructura de módulos vacíos según AGENTS.md 4.1 (telegram, hitl, audit, budget, security, memory, model-provider, integrations, tools).
- Dockerfile, `.nvmrc` (Node 24), tsconfig strict según AGENTS.md 3, ESLint + Prettier + gitleaks pre-commit.
- GitHub Actions: lint, typecheck, test, generate:contract, build de imagen.

Para `Jin_Executor`:
- NestJS separado con endpoint `/execute` stub (sin lógica de K8s todavía).
- Mismo tratamiento: OpenAPI export, Dockerfile, `.nvmrc`, CI propio.

Para `Jin_Web` y `Jin_CLI`:
- Repos con scaffold mínimo (Vite+React y Ink respectivamente), script `generate:api` (openapi-typescript contra el contrato de core), README, CI propio.

CRITERIOS DE ÉXITO:
- En cada repo por separado: `pnpm install && pnpm build && pnpm test` funciona desde cero.
- `pnpm dev` en core y executor levanta cada servicio con hot reload.
- CI verde en un PR draft de cada repo, de forma independiente.
- `pnpm generate:api` en web genera tipos válidos desde el contrato de core.
- Ningún `any` en el código generado.
- Ningún workspace de pnpm, ningún turbo.json, ningún paquete compartido.

RESTRICCIONES:
- Jin_Core no importa `dockerode` ni `@kubernetes/client-node`.
- Nada de lógica de negocio real — solo boilerplate.
- Actualiza `../Jin_Docs/STATUS.md`.

ANTES DE IMPLEMENTAR:
Propón la estructura de carpetas de cada repo y las dependencias iniciales. Espera aprobación.
```

### 2.2 [CLAUDE CODE] — HITL classifier + audit log (repo: Jin_Core)

```
Vas a implementar dos piezas críticas de seguridad: el HITL classifier y el audit log con hash chain. Trabajas en el repo `Jin_Core`.

ANTES DE ESCRIBIR CÓDIGO:
1. Lee `../Jin_Docs/docs/BLUEPRINT.md` sección 9 (HITL) completa. Léela dos veces.
2. Lee `../Jin_Docs/AGENTS.md` completo, especialmente secciones 5 y 6.
3. Lee `CLAUDE.md` sección 3 (áreas que lideras).
4. Verifica que Antigravity haya terminado la Fase 2.1 revisando `../Jin_Docs/STATUS.md`.

TU TAREA:
Implementar en `src/hitl/`:
- `types.ts` — enum de niveles `auto | notify | confirm | dual-confirm`.
- `registry.ts` (en `src/tools/`) — declaración estática de tools con su `hitlLevel`. Empieza con 3 tools stub: `readEmails` (auto), `createCalendarEvent` (notify), `sendEmail` (confirm).
- `classifier.ts` — función `classifyToolCall(toolName, inputs): HITLDecision`.
- `classifier.spec.ts` — tests cubriendo 100% de la matriz. Incluye tests que verifiquen que el LLM NO puede cambiar el level en runtime.
- `dual-confirm.service.ts` — máquina de estados para las dos aprobaciones separadas por 30s.
- `timeout.service.ts` — expiración de aprobaciones pendientes.

Implementar en `src/audit/`:
- Schema Prisma o Drizzle para tabla `audit_log` (ver BLUEPRINT 9.5).
- `hash-chain.service.ts` — cálculo y verificación de hash.
- `audit.service.ts` — API para registrar acciones.
- Tests de mutación: intentar modificar una row histórica rompe la chain.
- CronJob de verificación diaria de la chain.

CRITERIOS DE ÉXITO:
- Cobertura HITL: 100%.
- Cobertura audit chain: 100% incluyendo casos de mutación.
- Tests corren en <5s cada uno.
- Los DTOs públicos anotados para OpenAPI (web/cli generarán sus tipos desde el contrato — no crees paquetes compartidos).
- ADR en `../Jin_Docs/docs/adr/0001-hitl-four-levels.md` explicando la decisión de 4 niveles.

RESTRICCIONES:
- No implementes tools reales (Canvas, Google) todavía. Solo los stubs.
- No introduzcas librerías nuevas más allá de las ya aprobadas en `AGENTS.md`.
- Extended thinking activado para toda decisión no trivial.

USA CLAUDE OPUS 4.8 para esta tarea (es seguridad crítica).

ANTES DE IMPLEMENTAR:
Propón el diseño con firmas de funciones. Espera aprobación.
```

### 2.3 [CLAUDE CODE] — Executor con RBAC estricto (repo: Jin_Executor)

```
Vas a implementar el Executor: microservicio separado que ejecuta código LLM-generado en pods aislados. Trabajas en el repo `Jin_Executor` (es tuyo completo).

ANTES DE ESCRIBIR CÓDIGO:
1. Lee `../Jin_Docs/docs/BLUEPRINT.md` secciones 4.2, 4.3, 4.4 completas.
2. Lee `../Jin_Docs/AGENTS.md` sección 5.3 (separación de privilegios).
3. Confirma en `../Jin_Docs/STATUS.md` que la Fase 2.1 (scaffolding) está terminada.

TU TAREA:
Implementar en `src/`:
- `k8s.service.ts` — wrapper sobre `@kubernetes/client-node`. Métodos: `createPod`, `deletePod`, `waitForPod`, `getPodLogs`. Solo opera en namespace `agents-sandbox`.
- `rbac-validator.service.ts` — valida cada request contra whitelist de tools permitidas.
- `execute.controller.ts` — endpoint `POST /execute` con Zod validation. Recibe: `{tool, code, env, timeout, remote}`.
- `pod-lifecycle.service.ts` — ciclo de vida bajo demanda: crear pod al llegar la tarea, esperar readiness, ejecutar, recoger resultado, destruir. Sin warm pool. Verifica al startup que la imagen Deno está pre-pulled en el nodo y alerta si no.
- `modal.service.ts` — cliente para Modal API cuando `remote: true` (stub por ahora).
- Manifests Kubernetes (en el repo `Jin_Infra`, `k8s/base/executor/`): ServiceAccount, Role, RoleBinding, NetworkPolicies. Coordina ese PR aparte.

CRITERIOS DE ÉXITO:
- Tests de RBAC: peticiones para tools fuera de whitelist son rechazadas con 403.
- Tests de aislamiento: un pod en `agents-sandbox` no puede alcanzar servicios en `jin` (verificar con test de integration).
- Pod listo para ejecutar en <5 s con la imagen cacheada (medido desde la petición).
- Manifest de Role tiene solo permisos mínimos (create/get/list/delete pods en un namespace).
- `contracts/openapi.json` actualizado — core generará su cliente desde ahí.
- ADR: `../Jin_Docs/docs/adr/0002-executor-separation.md`.

RESTRICCIONES:
- Nunca ejecutes código sin timeout.
- Nunca crees pods con `privileged: true` o `hostNetwork`.
- El SDK de K8s solo existe en este repo, jamás en Jin_Core.

USA CLAUDE OPUS 4.8.

ANTES DE IMPLEMENTAR:
Propón el diseño de las interfaces HTTP y los manifests de RBAC. Espera aprobación.
```

### 2.4 [ANTIGRAVITY] — Telegram bot con grammY + ModelProvider (repo: Jin_Core)

```
Vas a implementar dos módulos: el bot de Telegram y el abstraction layer de modelos LLM. Trabajas en el repo `Jin_Core`.

ANTES DE ESCRIBIR CÓDIGO:
1. Lee `../Jin_Docs/docs/BLUEPRINT.md` secciones 6, 8.2.
2. Lee `../Jin_Docs/docs/MODEL_ROUTING.md` completo.
3. Lee `../Jin_Docs/AGENTS.md` completo.

TU TAREA A:
Implementar `src/model-provider/`:
- `types.ts` — interface `ModelProvider` uniforme.
- `anthropic.provider.ts` — implementación para Claude.
- `google.provider.ts` — implementación para Gemini.
- `router.service.ts` — selector según `TaskProfile` desde `config/models.yaml`.
- `failover.service.ts` — retry + fallback.
- `config/models.yaml` — el archivo con los profiles (ver MODEL_ROUTING.md sección 2.1).
- Tests con mocks de ambas APIs.

TU TAREA B:
Implementar `src/telegram/`:
- Bot con grammY, webhook mode.
- Comandos: `/start`, `/status`, `/tasks`, `/approve <id>`, `/reject <id>`, `/budget`.
- Cards de HITL con botones inline.
- Auth: solo owner (chat_id whitelisted en Infisical).
- Integración con el HITL classifier de Claude (Fase 2.2).

CRITERIOS DE ÉXITO:
- Puedes enviar un mensaje al bot y recibir respuesta pasando por el ModelProvider.
- Un flujo de HITL confirm funciona end-to-end en Telegram.
- Tests de failover: si Anthropic 429, cambia a Gemini automáticamente.
- Todos los tokens en Infisical, no en env.

RESTRICCIONES:
- Ningún model ID hardcoded fuera de `models.yaml`.
- Ningún dato de mensajes se logea en plain (privacidad).
- Coordina con Claude Code sobre la interface del HITL classifier antes de consumirlo (mismo repo: revisa STATUS.md).

ANTES DE IMPLEMENTAR:
Propón el plan. Espera aprobación.
```

---

## Fase 3 — Primera integración end-to-end

### 3.1 [ANTIGRAVITY] — Integración Canvas LMS (repo: Jin_Core)

```
Vas a integrar Canvas LMS usando la instancia de prueba del owner. Trabajas en el repo `Jin_Core`.

ANTES DE ESCRIBIR CÓDIGO:
1. Lee `../Jin_Docs/docs/BLUEPRINT.md` sección 7.1.
2. Lee `../Jin_Docs/AGENTS.md` completo.
3. Confirma con el owner: qué URL de Canvas usar, qué token, qué curso de prueba.
4. Lee la documentación oficial de Canvas API vía MCP: https://canvas.instructure.com/doc/api/

TU TAREA:
Implementar `src/integrations/canvas/`:
- `client.ts` — cliente REST con auth por Personal Access Token (desde Infisical).
- `tools/list-assignments.tool.ts` — tool `auto` para listar tareas próximas.
- `tools/get-course-content.tool.ts` — tool `auto` para leer materiales.
- `tools/schedule-study-block.tool.ts` — tool `notify` para crear bloque en Calendar (delega a Google Calendar integration cuando exista).
- `shadowing.cron.ts` — cron 00:00 local que:
  1. Consulta Canvas por cambios en las últimas 24 h.
  2. Indexa cambios en pgvector.
  3. Genera resumen con Gemini 3.1 Pro (contexto largo por PDFs).
  4. Envía notificación Telegram con acciones sugeridas (HITL confirm).
- Tests con nock/mocks del API de Canvas.

CRITERIOS DE ÉXITO:
- Puedes correr manualmente el shadowing y recibir un resumen coherente en Telegram.
- Todo input externo (contenido de Canvas) pasa por el injection sanitizer.
- El audit log registra cada tool call con `external_inputs_summary`.
- Rate limiting: máximo 30 req/min contra Canvas API.

RESTRICCIONES:
- Solo lectura por ahora. Nada de submits ni respuestas a profesores.
- Prohibido enviar contenido a APIs externas sin envolverlo en `<untrusted_content>`.

USA GEMINI 3.1 PRO para el análisis (contexto largo por PDFs).

ANTES DE IMPLEMENTAR:
Propón el plan. Especifica cómo probarás sin depender de la API real de Canvas.
```

---

## Fase 4 — Guardrails avanzados

### 4.1 [CLAUDE CODE] — Budget guard + kill switch (repo: Jin_Core)

```
Vas a implementar el sistema de budget: per-session, daily, y kill switch. Trabajas en el repo `Jin_Core`.

ANTES DE ESCRIBIR CÓDIGO:
1. Lee `../Jin_Docs/docs/BLUEPRINT.md` sección 9.6.
2. Lee `../Jin_Docs/AGENTS.md` completo.
3. Lee `CLAUDE.md`.

TU TAREA:
Implementar `src/budget/`:
- `budget.service.ts` — tracking de tokens y dólares por session, per-day, global.
- `guard.middleware.ts` — intercepta cada llamada al ModelProvider y verifica presupuesto antes.
- `kill-switch.service.ts` — detección de runaway (>2× consumo/hora vs 24h previas).
- `unpause.controller.ts` — endpoint para reanudar tras kill switch (requiere confirm HITL).
- Métricas Prometheus: `tokens_consumed_total`, `budget_remaining_ratio`, `runaway_detected_total`.
- Alertas en Alertmanager → Telegram cuando budget al 80% y 100%.
- Tests: simular consumos crecientes y verificar cortes.

CRITERIOS DE ÉXITO:
- Test de runaway simulado: en 1 min se consumen X tokens; el kill switch dispara.
- Test de degradación: al 80% del daily, las tareas nuevas usan Haiku en vez de Opus.
- El kill switch bloquea todas las tools salvo `unpause` (que es confirm HITL).
- Cobertura >95%.

USA CLAUDE OPUS 4.8.

ANTES DE IMPLEMENTAR:
Propón el diseño incluyendo cómo el guard se integra con el ModelProvider sin acoplar demasiado.
```

### 4.2 [ANTIGRAVITY] — Integración Google Calendar + Gmail (repo: Jin_Core)

```
Vas a integrar Google Calendar y Gmail con OAuth Testing. Trabajas en el repo `Jin_Core`.

ANTES DE ESCRIBIR CÓDIGO:
1. Lee `../Jin_Docs/docs/BLUEPRINT.md` sección 7.2.
2. Lee `../Jin_Docs/AGENTS.md` completo.
3. Confirma con el owner: qué scopes, qué cuenta usar.

TU TAREA:
Implementar `src/integrations/google/`:
- `oauth.service.ts` — flujo OAuth Testing, refresh cada 7 días con cron que avisa 24h antes.
- `calendar/` — cliente + tools (list events, create event, update, delete).
- `gmail/` — cliente + tools (list messages, get thread, send).
- Clasificación HITL:
  - listar, leer: `auto`.
  - crear evento: `notify`.
  - responder correo: `confirm`.
  - enviar correo nuevo: `confirm`.
  - borrar evento pasado: `notify`. Borrar evento futuro: `confirm`.
- Tests con mocks del SDK googleapis.

CRITERIOS DE ÉXITO:
- OAuth funciona end-to-end (con refresh manual documentado en runbook).
- Puedes desde Telegram: "resume mis correos de hoy" y recibir un summary.
- Puedes desde Telegram: "responde a Juan que sí, con más info sobre X" y el bot pide confirm con la card antes de enviar.
- Ningún scope innecesario (mínimos posibles).

RESTRICCIONES:
- Contenido de correos envuelto en `<untrusted_content>`.
- Nunca logea contenido en plain.
- Rate limiting según los límites de Google APIs.

ANTES DE IMPLEMENTAR:
Propón el plan.
```

### 4.3 [CLAUDE CODE] — Memoria extendida con sqlite-vec (repo: Jin_Core)

```
Vas a implementar la memoria extendida local del agente con sqlite-vec. Trabajas en el repo `Jin_Core`.

ANTES DE ESCRIBIR CÓDIGO:
1. Lee `../Jin_Docs/docs/BLUEPRINT.md` secciones 3.3.1 y 6.4.
2. Lee `../Jin_Docs/AGENTS.md` sección 5.1 (el contenido que entra a memoria también se sanitiza).
3. Lee `CLAUDE.md` sección 3 (src/memory es tuyo).

TU TAREA:
Implementar `src/memory/`:
- `memory.service.ts` — API pública: `recall(query, k, filters)`, `remember(entry)`, `consolidate(sessionId)`.
- `store.ts` — acceso a `memory.db` vía better-sqlite3 + extensión sqlite-vec. WAL mode. Un solo writer.
- Schema: tabla vec con embedding + columnas de metadata `tipo`, `fuente`, `fecha`, `modelo_embedding`.
- `consolidation.service.ts` — al cierre de cada sesión de agente, destila hechos/preferencias/lecciones (TaskProfile `memory_consolidation`) y los guarda.
- Recall al inicio de sesión: inyecta las k entradas más relevantes al contexto del agente.
- Tests: round-trip de embeddings, filtros de metadata, y que el contenido externo pasa por el sanitizer antes de persistir.

CRITERIOS DE ÉXITO:
- Una preferencia guardada en una sesión aparece en el recall de la siguiente (test E2E con mocks del LLM).
- Búsqueda KNN brute-force + filtros de metadata (NO dependas de los índices ANN de sqlite-vec: siguen en alpha).
- `memory.db` entra al backup nocturno (coordina con el script de Jin_Infra, Fase 1.2).
- Solo este módulo importa better-sqlite3/sqlite-vec en todo el repo.
- Cobertura >85%.

RESTRICCIONES:
- Los pods efímeros jamás acceden a memory.db.
- Nada de memoria "automática" de todo: solo lo que la consolidación destila explícitamente.

USA CLAUDE OPUS 4.8 para el diseño del schema y el flujo de consolidación.

ANTES DE IMPLEMENTAR:
Propón el schema y las firmas de la API. Espera aprobación.
```

---

## Fase 5+ — Contexto de realidad del código (leer antes de usar estos prompts)

Escritos el 2026-07-25, con las Fases 1-4 ya mergeadas. Desviaciones del BLUEPRINT ya decididas y documentadas en `STATUS.md` que estos prompts asumen (no las "re-arregles"):

- **No hay BullMQ** en ningún repo: todos los jobs usan `@nestjs/schedule` (`@Cron`). No lo introduzcas.
- **No hay Alertmanager**: las alertas van directo de `Jin_Core` al bot de Telegram (decisión del owner, Fase 4.1).
- **No existe agent loop todavía**: el handler de texto libre de Telegram llama `chat_conversational` sin tools. La Fase 5.1 lo construye — es el prerequisito de casi todo lo demás.
- **No existe auth ni API REST/WS** para interfaces: `Jin_Core` solo expone el webhook de Telegram y `/metrics`. La Fase 6.1 la construye.
- **Ya existe** el mecanismo "aprobar → ejecutar" (`ToolExecutorRegistry` + `ApprovalExecutionService`, `src/hitl/`, PR #8): las tools `confirm`/`dual-confirm` difieren su ejecución creando una pending approval con `payload`, y la ejecución real ocurre al aprobar desde Telegram. Todo tool nuevo con efectos reales se enchufa ahí, no inventa su propio flujo.
- **Ya existe** memoria extendida (`src/memory/`, PR #10) con `remember()`/`recall()`/`consolidate()` — sin conectar aún al ciclo de vida de sesiones (Fase 5.3).
- `Jin_Executor` tiene RBAC + ejecución aislada en pods Deno funcionando (testcontainers K3s real); **Modal es un stub** (`ModalNotImplementedError`) y **la tool `runCode` no está declarada** en `registry.ts`.
- `Jin_Web` y `Jin_CLI` son los templates de scaffolding **sin tocar** desde Fase 2.1.
- `Jin_Infra` tiene la base (Postgres, Redis, Traefik, cloudflared, observabilidad, backups, RBAC del Executor) pero **ningún manifest de deploy** para core/executor/web, y no hay GitOps.

Orden de dependencias: 5.1 → (5.2, 5.3, 5.4 en paralelo — 5.4 es de Claude Code y 5.3 de Antigravity, sin colisión) → 5.5 (tras 5.2: mismo repo Executor, mismo owner) → 6.1 → (6.2+6.3 en secuencia, ambas de Claude Code — mismo repo Jin_Web, misma regla de "una rama activa por repo" — en paralelo con 6.4 de Antigravity) → 7.1 → 7.2 → 7.3. Mientras Claude Code hace 5.4/5.5/6.1/6.2/6.3, Antigravity avanza 5.3 y luego 6.4; ambos convergen en 7.x.

**Nota (2026-08-02):** `Jin_Web` (Fases 6.2 y 6.3) se reasignó de Antigravity a Claude Code por decisión explícita del owner, no por el criterio habitual de "dividir por fortaleza" (ver `docs/WORKFLOW.md` §2.1). Los prompts de 6.2/6.3 abajo ya están actualizados con la etiqueta `[CLAUDE CODE]`; `Jin_CLI` (6.4) sigue siendo de Antigravity.

---

## Fase 5 — Agent loop + ejecución aislada real

### 5.1 [CLAUDE CODE] — Agent loop con tool-calling (repo: Jin_Core)

```
Vas a construir el agent loop: la pieza que convierte a Jin de "chat que responde texto" en un orquestador que INVOCA tools. Hoy ninguna tool se dispara desde el LLM — los handlers existen (Canvas, Google) pero nadie los llama. Trabajas en el repo `Jin_Core`, en `src/agent/` (área nueva, la lidera Claude Code: compone hitl + tools + model-provider + budget, todas áreas tuyas).

ANTES DE ESCRIBIR CÓDIGO:
1. Lee `../Jin_Docs/docs/BLUEPRINT.md` secciones 6 (Modelos y ruteo) y 9 (HITL) completas.
2. Lee `../Jin_Docs/AGENTS.md` sección 5 completa (seguridad — especialmente 5.1 y 5.4).
3. Lee en el código real: `src/hitl/classifier.ts`, `src/hitl/tool-executor.registry.ts`, `src/hitl/approval-execution.service.ts`, `src/hitl/dual-confirm.service.ts`, `src/tools/registry.ts`, `src/budget/budget-guarded-router.service.ts`, `src/model-provider/model-provider.types.ts`.
4. Revisa `../Jin_Docs/STATUS.md`.

TU TAREA:
- Extender `src/model-provider/` para soportar tool-use nativo de ambos vendors (Anthropic tool use / Gemini function calling): `ModelCompletionRequest` gana declaración de tools y la response puede ser texto o tool-calls. Mantener compatibilidad con todos los callers actuales.
- `src/agent/agent.service.ts` — el loop: LLM propone tool-call → `classifyToolCall()` decide el nivel → `auto`: ejecutar + `auditService.recordToolCall`; `notify`: ejecutar + auditar + notificar después; `confirm`/`dual-confirm`: `createPendingApproval({..., payload})` y el loop devuelve "en espera de aprobación" (la ejecución posterior ya existe: `ApprovalExecutionService`). El resultado de cada tool vuelve al contexto del LLM y el loop continúa hasta respuesta final o límite.
- Un registro de invocación directa para tools `auto`/`notify` (los handlers reales viven en `src/integrations/**`, no puedes importarlos desde `src/agent/` sin ciclos — mismo patrón `onModuleInit()` + registro que `ToolExecutorRegistry`; evalúa unificar ambos registros o mantenerlos separados, y justifica).
- **Ejecución autónoma orientada a objetivos (requisito del owner, 2026-07-25):** el loop no es solo "reaccionar al último resultado" — para tareas multi-paso el agente genera primero un plan explícito (lista de pasos, patrón plan-and-solve), lo persiste en el estado del turno, y va marcando pasos completados. Self-correction explícita: si una tool falla o el resultado no satisface el paso, el agente re-evalúa y reintenta con enfoque ajustado, con límite de reintentos por paso (configurable) además del límite global de iteraciones. El plan y su progreso son inspeccionables (se devuelven en la respuesta y quedan disponibles para las interfaces de Fase 6).
- Guardas duras: máximo de iteraciones por turno y de reintentos por paso (configurables), todo pasa por `BudgetGuardedModelRouter` con un `sessionId` compartido por turno, y todo resultado de tool que contenga contenido externo se envuelve con el sanitizer antes de volver al contexto.
- Tests: loop completo con LLM mockeado (auto ejecuta, confirm difiere y NO ejecuta, límite de iteraciones corta, fallo de tool → reintento ajustado → éxito, reintentos agotados → el agente reporta el fallo honestamente en vez de inventar éxito, inputs maliciosos en resultados de tools quedan envueltos).

CRITERIOS DE ÉXITO:
- Un turno simulado "borra mi evento X de mañana" produce una pending approval con payload correcto y NO borra nada.
- Un turno "lista mis eventos" ejecuta la tool auto, audita, y la respuesta final del LLM incorpora el resultado.
- El `hitlLevel` sale EXCLUSIVAMENTE de `classifyToolCall` — ningún camino donde el LLM lo influya.
- `npx tsc --noEmit -p tsconfig.json`, lint, tests y CI verdes. Cobertura >85% en `src/agent/`.
- Actualiza `../Jin_Docs/STATUS.md` al empezar y al terminar, documentando el contrato de registro para que Antigravity enchufe sus handlers (Fase 5.3).

RESTRICCIONES:
- NO conectes Telegram todavía (eso es 5.3, área de Antigravity). Expón `AgentService` desde un `AgentModule` y ya.
- NO toques los handlers de `src/integrations/**`.
- Prohibido un `switch(toolName)` hardcodeado en `src/agent/` — registro dinámico o nada.

USA CLAUDE OPUS 4.8 para el diseño del loop y del contrato de registro; Sonnet para los tests mecánicos.

ANTES DE IMPLEMENTAR: Propón el plan (diseño del loop, cambios a model-provider, decisión sobre los registros). Espera aprobación.
```

### 5.2 [CLAUDE CODE] — runCode end-to-end + Modal real (repos: Jin_Core y Jin_Executor)

```
Vas a cerrar la ejecución aislada: declarar la tool `runCode`, conectar Jin_Core con el Executor por HTTP interno, e implementar el cliente real de Modal (hoy `ModalNotImplementedError`). Ambos repos son tuyos (`registry.ts` en Core, Jin_Executor completo).

ANTES DE ESCRIBIR CÓDIGO:
1. Lee `../Jin_Docs/docs/BLUEPRINT.md` sección 4 completa (orquestación y ejecución aislada, especialmente 4.2-4.5).
2. Lee `../Jin_Docs/AGENTS.md` 5.3 (separación de privilegios) y 5.5 (whitelist de egresos).
3. Lee `../Jin_Docs/docs/adr/0003-*.md` y el código real de `Jin_Executor/src/execute/` y `src/modal/`.
4. Revisa `../Jin_Docs/STATUS.md`.

TU TAREA:
- `registry.ts`: declarar `runCode` con `hitlLevel: 'confirm'` (BLUEPRINT Fase 5 lo fija explícitamente) y `timeoutBehavior` justificado.
- Jin_Core: `src/integrations/executor/` NO — va en área tuya: cliente HTTP del Executor (`EXECUTOR_BASE_URL` requerida en env.schema, fail-fast) + registro del executor de `runCode` en `ToolExecutorRegistry` y en el registro de invocación del agent loop (5.1). El código de core JAMÁS importa clientes de Kubernetes (regla de oro #1).
- Jin_Executor: implementar `ModalService` real (SDK/API de Modal, token vía env desde Infisical, requerido fail-fast) para `remote: true` — target: código Python con dependencias (pandas). Decidir y documentar el criterio local-Deno vs Modal (lenguaje/deps/recursos).
- Tests: unit con Modal mockeado (API externa) + integración existente de pods Deno intacta. En core, tests del cliente HTTP mockeando el Executor.

CRITERIOS DE ÉXITO:
- Criterio del BLUEPRINT: "analiza este CSV con pandas" → pending approval → al aprobar, el Executor lo manda a Modal y el resultado vuelve al chat.
- Un `runCode` con código Deno simple corre en pod local con NetworkPolicy de egreso aplicada.
- CI verde en AMBOS repos (recuerda actualizar los `env:` de los workflows con las variables nuevas — ya nos pasó dos veces).
- Actualiza `../Jin_Docs/STATUS.md`.

RESTRICCIONES:
- El resultado de ejecución es contenido no confiable: se envuelve con el sanitizer antes de entrar a cualquier prompt.
- Límites de recursos y timeout duros en ambas rutas (local y Modal).

USA CLAUDE OPUS 4.8 (frontera de seguridad + diseño cross-repo).

ANTES DE IMPLEMENTAR: Propón el plan. Espera aprobación.
```

### 5.3 [ANTIGRAVITY] — Sesiones reales en Telegram: agent loop + memoria (repo: Jin_Core)

```
Vas a conectar dos piezas ya construidas al bot de Telegram (tu área, `src/telegram/**`): el agent loop (Fase 5.1) y la memoria extendida (`src/memory/`, Fase 4.3). Hoy el texto libre responde sin tools y sin memoria.

ANTES DE ESCRIBIR CÓDIGO:
1. Lee `../Jin_Docs/docs/BLUEPRINT.md` secciones 3.3.1 y 6.4.
2. Lee `../Jin_Docs/STATUS.md` — sección "Fase 4.3 resumen técnico" (handoff de memoria) y lo que Claude Code haya documentado del contrato del agent loop (5.1).
3. Lee las firmas reales: `src/agent/` (AgentService), `src/memory/memory.service.ts` (recall/consolidate). NO modifiques nada dentro de esas carpetas.

TU TAREA:
- Registrar los handlers de tus integraciones (Canvas, Google Calendar, Gmail — los `auto`/`notify`) en el registro de invocación del agent loop, en el `onModuleInit()` de cada módulo (mismo patrón que ya usaste con `ToolExecutorRegistry` para `sendEmail`/`deleteCalendarEventFuture`).
- El handler de texto libre pasa a llamar `AgentService` en vez de `budgetGuardedRouter.complete()` directo.
- Ciclo de sesión: al primer mensaje tras X minutos de inactividad se abre sesión nueva (uuid) e inyecta `recall(mensaje, k)` al contexto; al cerrarse (inactividad vía cron y/o comando `/endsession` — propón y justifica) se llama `consolidate(sessionId, transcript)`. Definir qué es el "transcript" (buffer en memoria de la sesión) y su límite de tamaño.
- Comando `/memory <query>` para que el owner consulte la memoria a mano.
- Tests actualizados en `telegram-bot.service.spec.ts` + tests del ciclo de sesión.

CRITERIOS DE ÉXITO:
- Flujo real: "mándale un correo a X" por Telegram → pending approval con payload → `/approve` → correo enviado (mecanismo PR #8, ya existente).
- Una preferencia dicha en una sesión aparece en el contexto de la siguiente (test con LLM y embeddings mockeados).
- CI verde, verificado con `gh pr checks` (no solo local).
- Actualiza `../Jin_Docs/STATUS.md`.

RESTRICCIONES:
- NO toques `src/agent/**`, `src/memory/**`, `src/hitl/**` — si el contrato no te alcanza, pide el cambio vía STATUS.md.
- El contenido de mensajes entrantes sigue las reglas de sanitización de siempre.

ANTES DE IMPLEMENTAR: Propón el plan, incluyendo tu decisión de cierre de sesión. Espera aprobación.
```

### 5.4 [CLAUDE CODE] — Orquestación multi-agente: sub-agentes coordinados (repo: Jin_Core)

```
Vas a extender el agent loop (5.1, ya mergeado) para que el orquestador pueda descomponer un objetivo y delegarlo a DOS O MÁS sub-agentes que trabajan en paralelo y se coordinan sin discrepar. Requisito explícito del owner (2026-07-25). Es área de Claude Code (`src/agent/`) y requiere ADR nuevo: cambia el modelo de ejecución del sistema.

DECISIÓN ARQUITECTÓNICA YA TOMADA POR EL OWNER (no la re-litigues, documentala en el ADR):
Los sub-agentes NO se comunican directamente entre sí (peer-to-peer, DMs). Se coordinan vía un "task ledger" compartido estilo Jira, mediado por el orquestador: el orquestador descompone el objetivo en tickets (descripción, estado, agente asignado, dependencias), cada sub-agente lee el tablero COMPLETO (qué hicieron los demás, qué decisiones ya se tomaron) y escribe en sus propios tickets: resultados, observaciones y comentarios — incluyendo señalar contradicciones con lo de otro agente. Las discrepancias NO se suprimen: se registran como conflicto visible en el ticket, y las resuelve una escalera de decisión explícita — (a) conflicto de bajo riesgo: el orquestador decide en modo-auto con el tablero completo como contexto, dejando registrada la resolución y el porqué; (b) conflicto material (afecta algo de nivel confirm, o los agentes proponen acciones incompatibles): se escala al OWNER vía HITL — el owner es siempre el nivel máximo de la escalera. Razón del diseño: es la diferencia entre un equipo que discute dentro del ticket (trazable, el lead decide, todos ven el acuerdo) y uno que se manda mensajes privados (nadie sabe qué se acordó ni por qué). Funcionalmente los agentes sí se responden entre sí — a través del tablero.

ANTES DE ESCRIBIR CÓDIGO:
1. Lee `../Jin_Docs/docs/BLUEPRINT.md` secciones 6 y 9, y los ADRs existentes en `../Jin_Docs/docs/adr/`.
2. Lee el código real de `src/agent/` (el loop de 5.1 es la unidad que vas a instanciar N veces), `src/budget/budget-guarded-router.service.ts`, `src/audit/audit.service.ts`.
3. Revisa `../Jin_Docs/STATUS.md`.

TU TAREA:
- **El ledger se persiste en Postgres** (tablas nuevas + migración con `.down.sql`, patrón de siempre): tickets con estado (pending/in-progress/done/failed/blocked), agente asignado, dependencias, y un hilo de comentarios por ticket (autor = agente u orquestador u owner). Persistido, no en memoria: un objetivo puede quedar bloqueado días esperando una aprobación HITL y debe sobrevivir restarts — mismo criterio que `pending_approvals`. Además, las interfaces de Fase 6 van a renderizar este tablero (el dashboard lo muestra como un board estilo Jira).
- `src/agent/orchestrator.service.ts` — descompone el objetivo en tickets, asigna cada uno a un sub-agente, resuelve el orden por dependencias (paralelo cuando no hay dependencia), y al final reconcilia los resultados en una respuesta única aplicando la escalera de decisión de arriba: conflictos de bajo riesgo los resuelve él (registrando resolución y porqué en el ticket), conflictos materiales escalan al owner vía HITL.
- **Trabajo sobre código en branches seguras (requisito del owner):** cuando una tarea implica modificar un repo, el sub-agente trabaja SIEMPRE en una branch propia (`feature/agent/<ticket>`), jamás en main. El merge a main es una tool con `hitlLevel: 'confirm'` — lo aprueba el owner. Declarala en `registry.ts`.
- Sub-agente = instancia del agent loop de 5.1 con: (a) subset acotado de tools (el orquestador decide cuáles según la tarea — nunca más de las necesarias), (b) `sessionId` propio derivado del turno padre (el budget guard ve a cada sub-agente y el cap diario/kill switch aplican globalmente), (c) sus propios límites de iteraciones/reintentos, (d) acceso de LECTURA al ledger y escritura solo de sus propias entradas.
- HITL intacto: un tool `confirm`/`dual-confirm` invocado por un sub-agente crea la misma pending approval de siempre, con atribución de QUÉ sub-agente y QUÉ tarea del ledger la originó (visible en el planSummary y en el audit log — extiende `actor` o agrega metadata, justifica en el ADR). El sub-agente queda bloqueado en esa tarea hasta resolución; el orquestador puede seguir con tareas independientes.
- Audit: cada acción registra qué agente la ejecutó. El ledger completo del turno queda inspeccionable (para las interfaces de Fase 6).
- Escribir el ADR (siguiente número libre) con el diseño completo.
- Tests: descomposición con dependencias respeta el orden; tareas independientes corren en paralelo; contradicción entre sub-agentes dispara reconciliación; un confirm de un sub-agente NO bloquea tareas independientes; el kill switch a mitad de un turno multi-agente detiene TODO; presupuesto agregado del turno = suma de sub-agentes.

CRITERIOS DE ÉXITO:
- Un objetivo simulado tipo "revisa mi correo y mi calendario y proponme la agenda de mañana" se descompone en 2+ tareas, corre con 2 sub-agentes coordinados por el ledger, y produce UNA respuesta coherente sin contradicciones.
- Ninguna vía por la que un sub-agente ejecute algo confirm sin aprobación, ni por la que dos sub-agentes se manden instrucciones directamente.
- Cobertura >85% en lo nuevo. CI verde en real. Actualiza `../Jin_Docs/STATUS.md`.

RESTRICCIONES:
- Máximo de sub-agentes concurrentes configurable (default bajo: 3). Profundidad de delegación = 1 (un sub-agente NO puede crear más sub-agentes — evita árboles runaway; si algún día hace falta, es otro ADR).
- Todo lo de 5.1 sigue aplicando: sanitización, hitlLevel estático, budget, límites.

USA CLAUDE OPUS 4.8 (diseño de concurrencia + modelo de seguridad).

ANTES DE IMPLEMENTAR: Propón el plan y el borrador del ADR. Espera aprobación.
```

### 5.5 [CLAUDE CODE] — Servicios de larga vida: preview apps con puertos expuestos (repos: Jin_Executor + Jin_Core + Jin_Infra)

```
Requisito del owner (2026-07-25): los agentes deben poder LEVANTAR apps corriendo — un frontend con `npm run dev`, un backend de prueba — no solo ejecutar código run-to-completion. Hoy el Executor solo soporta pods que corren y mueren. Vas a agregar la categoría "pod de servicio": proceso de larga vida con puerto expuesto. Requiere ADR (amplía el RBAC del Executor y el modelo de exposición).

ANTES DE ESCRIBIR CÓDIGO:
1. Lee `../Jin_Docs/docs/BLUEPRINT.md` secciones 4 (ejecución aislada), 5.2 (separación de dominios — la regla de oro #10 es la base de este diseño) y `AGENTS.md` 5.5 (egress whitelist).
2. Lee el código real de `Jin_Executor/src/pod-lifecycle/` y `src/k8s/`, y los manifests de `Jin_Infra/k8s/base/executor/` (RBAC actual) y del wildcard `*.jinserver.com`.
3. Revisa `../Jin_Docs/STATUS.md`. Requiere 5.2 mergeado (mismo repo, mismo owner — secuencial).

TU TAREA:
- Jin_Executor: nueva capacidad `startService` / `stopService` — crea en `agents-sandbox` un pod de larga vida + Service + IngressRoute de Traefik bajo `https://<slug>.jinserver.com` (NUNCA `jeanfranck.com` ni ninguno de sus subdominios — regla de oro #10: esto ES contenido generado por agentes). El pod corre el comando declarado (`npm run dev`, `node server.js`, etc.) sobre un workspace montado (el código que el agente generó/clonó en su branch).
- **Slug con entropía**: `<nombre-legible>-<sufijo aleatorio>` (ej. `residuos-educan-a7f3k9`). Delante de una preview no hay login, así que la única barrera contra quien adivine la URL es la parte aleatoria; el prefijo legible existe solo para distinguirlas de un vistazo. Nunca un slug derivado únicamente del nombre del proyecto.
- **TTL obligatorio**: el owner lo define al aprobar (ej. "mantenlo 4h"), con default corto y **cap duro de 24h**. Un reaper (`@Cron`) destruye pod+Service+IngressRoute vencidos. Extender el TTL requiere una aprobación nueva, no es un flag. Límite de servicios concurrentes (default bajo).
- **Restricciones de la VM (verificadas contra los manifests reales)**: el namespace `agents-sandbox` ya tiene `ResourceQuota` de 6Gi/1500m y `LimitRange` de 512Mi por pod (máx 2Gi) — respetarlos, no ampliarlos en este PR. Tres cosas que hay que resolver o el hot reload falla: (a) subir `fs.inotify.max_user_watches` en el nodo, porque al agotarse el hot reload deja de funcionar **en silencio**, sin error claro; (b) la VM es ARM64 (Ampere), así que las imágenes y dependencias nativas deben tener build aarch64 — declarárselo explícitamente al agente que genera el proyecto; (c) un PVC compartido como store de pnpm para no pagar 30-90s de `install` en frío por cada preview.
- Sin secretos: los pods de servicio jamás reciben env vars de Infisical ni tokens. NetworkPolicy: mismo modelo de egress whitelist que los pods de ejecución (npm registry para instalar deps, y nada más por default). Resources y probes obligatorios.
- RBAC: el ServiceAccount `executor` gana `create/delete` de Services e IngressRoutes SOLO en `agents-sandbox` — actualiza los manifests en Jin_Infra (coordina en STATUS.md: repo de Antigravity, cambio acotado tuyo por ser frontera de seguridad, mismo precedente que backups en Fase 1.2).
- Jin_Core: declarar las tools en `registry.ts` — `startPreviewService` con `hitlLevel: 'confirm'` (expone código de agentes a internet, aunque sea en el dominio sandbox), `stopPreviewService` con `notify`, `listPreviewServices` con `auto`. Registrar los executors correspondientes (cliente HTTP del Executor de 5.2).
- Tests: integración con el testcontainer K3s existente (pod de servicio arranca, responde HTTP dentro del clúster, el reaper lo destruye al vencer TTL), RBAC no permite crear Services fuera de agents-sandbox, unit tests del cliente en core.

CRITERIOS DE ÉXITO:
- Flujo completo: el agente propone "levanto el frontend que generé" → pending approval → al aprobar, el servicio queda accesible en `https://<slug>.jinserver.com` con hot reload funcionando → al vencer el TTL desaparece solo, con notificación.
- Ningún camino por el que un servicio quede corriendo sin TTL, reciba un secreto, o se exponga bajo `jeanfranck.com` o cualquier subdominio suyo.
- CI verde en real en los repos tocados. ADR escrito. Actualiza `../Jin_Docs/STATUS.md`.

RESTRICCIONES:
- El Executor sigue siendo el ÚNICO que habla con Kubernetes (regla de oro #1).
- Prohibido `hostNetwork`, prohibido `latest`, prohibido montar el socket de nada.

USA CLAUDE OPUS 4.8 (frontera de seguridad: exposición de código de agentes a internet).

ANTES DE IMPLEMENTAR: Propón el plan y el borrador del ADR. Espera aprobación.
```

---

## Fase 6 — API pública e interfaces

### 6.1 [CLAUDE CODE] — Auth + API REST/WebSocket para interfaces (repo: Jin_Core)

```
Vas a construir la puerta de entrada para el dashboard y la CLI: auth single-user con JWT y la API REST + WebSocket que exponen lo que hoy solo se ve por Telegram. Es la frontera de seguridad del sistema completo — la lidera Claude Code.

ANTES DE ESCRIBIR CÓDIGO:
1. Lee `../Jin_Docs/docs/BLUEPRINT.md` secciones 4.1 (auth), 4.6 (contratos OpenAPI), 5 (dominios) y 9.7 (rate limiting).
2. Lee `../Jin_Docs/AGENTS.md` secciones 4.5 (contratos entre repos) y 5 completa.
3. Revisa `src/generate-contract.ts` y `contracts/openapi.json` actuales.

TU TAREA:
- `src/auth/`: login single-user (password del owner verificada contra hash en env/Infisical, nunca en claro), emisión de JWT firmado (llave vía Infisical), guard global de NestJS — TODO endpoint protegido por defecto, allowlist explícita para login/webhook de Telegram/metrics/health.
- Endpoints REST (los datos ya existen, solo se exponen): aprobaciones pendientes + aprobar/rechazar (vía `ApprovalExecutionService` — mismo camino que Telegram, sin duplicar lógica), audit log paginado, estado de budget/kill switch (+ unpause), memoria (`recall`), y chat contra `AgentService`.
- WebSocket gateway para eventos en vivo: nueva pending approval, alerta de budget, respuesta de chat en streaming si el modelo lo permite.
- Rate limiting en la API pública (BLUEPRINT 9.7).
- Regenerar `contracts/openapi.json` — Web y CLI consumirán tipos generados, jamás copiados a mano (regla de oro #11).
- Tests: guards (sin token → 401, token inválido → 401), cada endpoint con servicios mockeados, e2e de login→acción protegida.

CRITERIOS DE ÉXITO:
- `curl` sin token a cualquier endpoint de datos → 401. Con token → funciona.
- Aprobar desde la API ejecuta la tool igual que `/approve` en Telegram (mismo servicio, test que lo pruebe).
- Contrato OpenAPI regenerado y commiteado; CI verde en real.
- Actualiza `../Jin_Docs/STATUS.md` documentando la API para Antigravity (6.2-6.4).

RESTRICCIONES:
- Single-user real: sin registro, sin roles, sin multi-tenancy (decisión del owner 2026-07-23).
- Ningún secreto nuevo opcional en env.schema — todo requerido fail-fast, y actualiza el CI workflow en el mismo PR.

USA CLAUDE OPUS 4.8 (auth es auditoría de seguridad por definición).

ANTES DE IMPLEMENTAR: Propón el plan. Espera aprobación.
```

### 6.2 [CLAUDE CODE] — Web Dashboard MVP (repo: Jin_Web)

```
Vas a construir el dashboard real de Jin — hoy el repo es el template de Vite sin tocar. Repo tuyo (reasignado de Antigravity a Claude Code el 2026-08-02, decisión explícita del owner — ver `../Jin_Docs/docs/WORKFLOW.md` §2.1).

ANTES DE ESCRIBIR CÓDIGO:
1. Lee `../Jin_Docs/docs/BLUEPRINT.md` secciones 5 (dominios: el dash vive en `jin.jeanfranck.com`) y 8 (interfaces).
2. Lee `../Jin_Docs/AGENTS.md` — énfasis en 4.5 (contratos) y las reglas de oro #8, #10, #11.
3. Lee la API real recién construida (6.1): `Jin_Core/contracts/openapi.json` y la nota de Claude Code en `STATUS.md`.

TU TAREA (MVP — Monaco y applets van en 6.3, NO acá):
- Setup real: router, `pnpm generate:api` consumiendo el contrato de core (tipos generados, cero tipos copiados), cliente HTTP con el JWT, pantalla de login.
- **Bandeja HITL**: lista viva de pending approvals (WebSocket), y cada card muestra tool, plan summary, payload legible y la lista de inputs externos que influyeron (regla de oro #8 — no es opcional). Botones aprobar/rechazar; dual-confirm muestra el temporizador de 30s.
- **Chat** con el agente (streaming si la API lo da).
- **Audit log** paginado con filtros; **panel de budget** (consumo diario, kill switch + unpause).
- Tests de componentes para la card HITL y el flujo de login.

CRITERIOS DE ÉXITO:
- Flujo completo contra core corriendo local: login → chat pide enviar un correo → la card aparece en vivo → aprobar → el resultado llega al chat.
- Cero `any`, cero tipos duplicados del backend.
- CI verde en real (`gh pr checks`).
- Actualiza `../Jin_Docs/STATUS.md`.

RESTRICCIONES:
- No introduzcas librerías de estado/UI pesadas sin proponerlas antes en el plan.
- El JWT no se guarda en localStorage sin justificar la decisión frente a alternativas en el plan.

USA CLAUDE OPUS 4.8 para las decisiones de arquitectura (rutas, manejo de estado, cliente WS, dónde vive el JWT); Sonnet 5 para componentes rutinarios una vez decidido el esqueleto.

ANTES DE IMPLEMENTAR: Propón el plan (estructura de rutas, manejo de estado, librerías). Espera aprobación.
```

### 6.3 [CLAUDE CODE] — Editor Monaco + applets (repo: Jin_Web)

```
Segunda mitad del dashboard (requiere 6.2 mergeado): edición de código con comentarios de la IA y applets generados por agentes. Repo tuyo (reasignado de Antigravity a Claude Code el 2026-08-02).

ANTES DE ESCRIBIR CÓDIGO:
1. Lee `../Jin_Docs/docs/BLUEPRINT.md` sección 8 (Zone Widgets, applets) y 5.2 (separación de dominios).
2. Regla de oro #10: contenido generado por agentes se sirve SOLO desde `jinserver.com`.

TU TAREA:
- Monaco embebido con Zone Widgets: comentarios inline de la IA línea por línea (pedidos vía la API de chat/agente), aceptar/descartar sugerencia por widget.
- Vista de applets: iframes sandboxeados (`sandbox` estricto + CSP) apuntando únicamente a `jinserver.com`.
- Tests de los componentes nuevos.

CRITERIOS DE ÉXITO:
- Criterio del BLUEPRINT Fase 6: editar código en el dashboard, la IA comenta línea por línea, y se aprueban cambios desde ahí.
- Ningún iframe puede apuntar a un origen fuera de `jinserver.com` (test que lo pruebe).
- CI verde en real. Actualiza `../Jin_Docs/STATUS.md`.

USA CLAUDE OPUS 4.8 para el diseño de los Zone Widgets y el aislamiento de los iframes (frontera de seguridad de dominios); Sonnet 5 para el resto.

ANTES DE IMPLEMENTAR: Propón el plan. Espera aprobación.
```

### 6.4 [ANTIGRAVITY] — CLI con Ink (repo: Jin_CLI)

```
Vas a construir la CLI real — hoy es el template de Ink sin tocar. Repo tuyo. Requiere 6.1 mergeado.

ANTES DE ESCRIBIR CÓDIGO:
1. Lee `../Jin_Docs/docs/BLUEPRINT.md` sección 8 (interfaces) y `AGENTS.md` 4.5.
2. Consume el contrato OpenAPI de core (tipos generados, igual que Web).

TU TAREA:
- `jin login` (guarda el token de forma segura en el keychain del OS o archivo con permisos 600 — justifica), `jin status` (budget, kill switch, salud), `jin tasks` (pending approvals) + `jin approve/reject <id>`, `jin chat` (sesión interactiva con el agente), `jin memory <query>`.
- Tests con la API mockeada.

CRITERIOS DE ÉXITO:
- `jin tasks` + `jin approve` completan el mismo flujo HITL que Telegram y el dashboard.
- CI verde en real. Actualiza `../Jin_Docs/STATUS.md`.

RESTRICCIONES:
- Cero tipos copiados a mano del backend (regla de oro #11).

ANTES DE IMPLEMENTAR: Propón el plan. Espera aprobación.
```

---

## Fase 7 — Deploy real, activación y hardening

### 7.1 [ANTIGRAVITY] — Deploy de las apps + GitOps (repos: Jin_Infra + Dockerfiles en cada repo)

```
La infra base existe (Fase 1) pero NINGUNA app tiene manifest de deploy — core, executor y web nunca se han desplegado. Vas a cerrar ese hueco. Jin_Infra es tuyo; los Dockerfiles se agregan en cada repo de app (coordina en STATUS.md los de Jin_Core/Executor, que comparten ownership de CI con Claude Code).

ANTES DE ESCRIBIR CÓDIGO:
1. Lee `../Jin_Docs/docs/BLUEPRINT.md` secciones 4.4 (resources obligatorios), 5 (dominios/Traefik) y 12 completa (deployment, GitOps con Flux, CI/CD).
2. Lee los manifests existentes en `Jin_Infra/k8s/base/` para seguir el estilo (probes, resources, kustomize).

TU TAREA:
- Dockerfiles multi-stage pinneados para Jin_Core, Jin_Executor y Jin_Web (build estático servido por nginx/caddy), publicados a GHCR desde el CI de cada repo.
- Manifests: Deployment de core (1 réplica, rolling con surge, PVC para `memory.db` montado en `MEMORY_DB_PATH`, secrets vía Infisical), Deployment del executor (ServiceAccount `executor` ya existente), Deployment de web, IngressRoutes de Traefik (`jin.jeanfranck.com` con el path `/api` ruteado al core vía route rule de Cloudflare, y `*.jinserver.com` para applets/previews).
- Flux bootstrap: GitOps sobre `Jin_Infra` (BLUEPRINT 12.1) — un push a main reconcilia el clúster.
- NetworkPolicies coherentes con las existentes.

CRITERIOS DE ÉXITO:
- `kubectl apply --dry-run=server` limpio en todo; `kubectl kustomize` sin errores.
- CI de cada repo publica imagen taggeada por SHA; Flux la despliega al actualizar el manifest.
- Ningún secreto en YAML; resources y probes en todo Deployment (AGENTS 4.4).
- Actualiza `../Jin_Docs/STATUS.md`.

RESTRICCIONES:
- Prohibido `latest`. Prohibido `hostNetwork`. El executor mantiene su RBAC actual sin ampliaciones.

ANTES DE IMPLEMENTAR: Propón el plan. Espera aprobación.
```

### 7.2 [CLAUDE CODE] — Runbook de activación real (repo: Jin_Infra, docs + scripts de apoyo)

```
Todo lo construido corre con credenciales falsas en CI. Vas a escribir el runbook + scripts de apoyo para la puesta en marcha REAL — los pasos manuales los ejecuta el owner, tu trabajo es que sean imposibles de hacer mal.

ANTES DE ESCRIBIR CÓDIGO:
1. Lee `../Jin_Docs/docs/BLUEPRINT.md` secciones 7 (integraciones) y 11 (secretos).
2. Revisa TODAS las env vars requeridas reales: `Jin_Core/src/config/env.schema.ts` y el equivalente del Executor.

TU TAREA:
- `docs/runbooks/activation.md` en Jin_Infra: checklist ordenado y verificable — bootstrap de la VM OCI (si no está hecho), carga de secretos reales en Infisical (lista exacta de claves), obtención del `GOOGLE_REFRESH_TOKEN` vía consent flow (con un script helper `scripts/google-oauth-bootstrap.ts` que imprime la URL de consentimiento y captura el token), alta del webhook real de Telegram, verificación de backups reales subiendo a R2, y smoke test E2E final.
- El smoke test E2E como checklist ejecutable: mensaje por Telegram → tool auto → correo con confirm → aprobar → verificar audit log y presupuesto → `/google-oauth-refreshed` → verificar dashboard y CLI con el mismo flujo.
- Runbook "VM perdida → reconstrucción desde IaC + backups" (criterio del BLUEPRINT Fase 7).

CRITERIOS DE ÉXITO:
- El owner puede activar el sistema completo siguiendo el runbook sin preguntar nada.
- Cada paso tiene su verificación ("cómo sé que funcionó").
- Actualiza `../Jin_Docs/STATUS.md`.

ANTES DE IMPLEMENTAR: Propón el plan. Espera aprobación.
```

### 7.3 [CLAUDE CODE] — Hardening final: chaos, auditoría, MCP (repos: varios)

```
Cierre del roadmap: el sistema debe aguantar 7 días autónomo sin intervención salvo aprobaciones HITL (criterio del BLUEPRINT Fase 7).

ANTES DE ESCRIBIR CÓDIGO:
1. Lee `../Jin_Docs/docs/BLUEPRINT.md` secciones 6.4 (MCP servers), 13 (testing) y 15 (reglas de oro).
2. Lee `../Jin_Docs/AGENTS.md` sección 5 completa — es tu checklist de auditoría.

TU TAREA:
- **Auditoría de seguridad completa**: verificar cada regla de oro (#1-#12) contra el código real con evidencia (test o referencia), intentos activos de prompt injection contra el agent loop (payloads en correos/eventos de Canvas que intenten escapar del sanitizer o escalar HITL), revisión de RBAC/NetworkPolicies. Entregable: informe en `Jin_Docs/docs/security-audit-fase7.md` con hallazgos y fixes.
- **Chaos tests** (Jin_Infra `scripts/chaos/`): matar el pod de core a mitad de un dual-confirm (el estado sobrevive — está en Postgres), tirar Postgres y verificar fail-ruidoso sin corrupción del hash chain, simular runaway real y verificar kill switch + alerta.
- **MCP servers** para docs externas (Context7 u oficiales) en la config del agente — sin pipeline propio de scraping.
- Runbooks de operación pendientes.

CRITERIOS DE ÉXITO:
- Informe de auditoría sin hallazgos críticos abiertos.
- Los 3 chaos tests pasan y quedan documentados.
- Criterio final del BLUEPRINT: 7 días autónomo (se valida en operación real post-activación).
- Actualiza `../Jin_Docs/STATUS.md`.

USA CLAUDE OPUS 4.8 (auditoría de seguridad).

ANTES DE IMPLEMENTAR: Propón el plan. Espera aprobación.
```

---

## Fase 8 — Prerequisitos de producción (auditoría de cobertura 2026-08-05)

> Estas dos fases salieron de auditar `BLUEPRINT.md` completo contra el código real: son funciones **especificadas en el blueprint y nunca construidas** que condicionan las Fases 7.2/7.3. **Orden de ejecución real: 7.1 → 8.1 → 7.2 → 8.2 → 7.3.** El número es una etiqueta, no un orden (ya hay precedente: 3.1 se ejecutó antes que 2.4).

### 8.1 [CLAUDE CODE] — Infisical SDK en runtime (repos: Jin_Core + Jin_Executor)

```
BLUEPRINT §11 dice literal: "core y Executor los cargan al startup vía SDK; nunca están en env plain". Hoy es exactamente al revés: todo pasa por `src/config/env.schema.ts` como env vars planas, y el SDK de Infisical no está importado en ningún lado (verificado: cero ocurrencias en `src/`). El manifest de Infisical SÍ existe y se despliega (Jin_Infra `k8s/base/infisical`, Fase 1.1) — lo que falta es que las apps lo consuman.

VA ANTES DE 7.2 A PROPÓSITO: la Fase 7.2 escribe el runbook de "carga de secretos reales en Infisical". Si ese runbook se escribe mientras el runtime sigue leyendo env plano, documenta el mecanismo equivocado y hay que reescribirlo entero.

ANTES DE ESCRIBIR CÓDIGO:
1. Lee `../Jin_Docs/docs/BLUEPRINT.md` §11 completa y `../Jin_Docs/AGENTS.md` §5.2.
2. Lee `Jin_Core/src/config/env.schema.ts` — es hoy la única fuente de verdad de las 16 variables requeridas, todas fail-fast. Ese contrato de "faltar una variable = no arranca" NO se negocia; lo que cambia es de dónde salen los valores.
3. Revisa `Jin_Infra/k8s/base/infisical/` y `Jin_Infra/scripts/bootstrap/02-seed-secrets.sh` (ya existe, ver qué asume).

TU TAREA:
- Cargar secretos desde Infisical al startup, ANTES de que corra `validateEnv()`, manteniendo el fail-fast intacto (si Infisical no responde o falta una clave, el proceso muere ruidoso — jamás arranca a medias).
- **Env plano sigue siendo el camino de desarrollo local**: el `.env` + `docker-compose.dev.yaml` no se rompen. Criterio explícito de cuándo usa cada uno (ej. `INFISICAL_ENABLED` o `NODE_ENV=production`), documentado en `.env.example`.
- Mismo tratamiento en `Jin_Executor` (tiene su propio `env.schema`).
- Tests: arranque con Infisical mockeado devolviendo el set completo, y arranque fallando ruidoso cuando falta una clave o el servicio no responde.

CRITERIOS DE ÉXITO:
- Ningún secreto real necesita estar en un env var del Deployment para que core/executor arranquen en producción.
- `pnpm dev` local sigue funcionando con `.env` sin Infisical corriendo.
- CI verde en real (`gh pr checks`). Actualiza `../Jin_Docs/STATUS.md`.

RESTRICCIONES:
- No debilites `env.schema.ts`: nada de volver opcional una variable "porque ahora viene de Infisical".
- El token/credencial de Infisical es el único secreto que sí vive fuera de Infisical (problema de bootstrap) — documentá dónde vive y por qué.

USA CLAUDE OPUS 4.8 (gestión de secretos = frontera de seguridad).

ANTES DE IMPLEMENTAR: Propón el plan. Espera aprobación.
```

### 8.2 [CLAUDE CODE] — Golden set de prompt injection (repo: Jin_Core)

```
BLUEPRINT §13.1 lista como **OBLIGATORIO**: "Prompt injection golden set: corpus de ~50 prompts adversariales conocidos; el sistema debe manejarlos correctamente". Verificado: no existe (cero archivos golden/adversarial en el repo). Hoy `injection-sanitizer.spec.ts` cubre el escapado de delimitador y el nonce, que es la unidad — no el sistema end-to-end contra un corpus real.

RELACIÓN CON 7.3: la Fase 7.3 incluye "intentos activos de prompt injection contra el agent loop" como parte de la auditoría. Eso es un ejercicio puntual; esto es la **suite de regresión permanente** que evita que un refactor futuro reabra un agujero ya cerrado. Hacé esta primero para que 7.3 audite contra una base ya cubierta.

ANTES DE ESCRIBIR CÓDIGO:
1. Lee `../Jin_Docs/AGENTS.md` §5.1 completa y `../Jin_Docs/docs/adr/` (ADR 0004, el porqué del nonce por sesión).
2. Lee `src/security/injection-sanitizer.ts` (incluida `summarizeUntrustedSources`, agregada 2026-08-04) y `src/agent/agent.service.ts` (dónde se envuelve el resultado de cada tool).

TU TAREA:
- Corpus de ~50 payloads adversariales versionado en el repo (`src/security/golden-set/`), cada uno con su categoría y su expectativa. Cubrí al menos: cierre de delimitador falso, tags con nonce inventado o ausente, instrucciones en otro idioma, payload en base64/rot13, "ignora tus instrucciones previas", intento de escalar el hitlLevel de una tool, intento de que el agente invente que una aprobación ya ocurrió, y contenido que imita el formato de un tool_result del sistema.
- Los tests corren contra el pipeline REAL (sanitizer + system prompt + clasificador HITL), con el LLM mockeado — lo que se verifica es que el contenido hostil llega neutralizado y que ninguna tool sube de nivel, no que el modelo "se porte bien".
- Documentá en el propio corpus qué ataque representa cada entrada: un golden set sin explicación es imposible de mantener.

CRITERIOS DE ÉXITO:
- Los ~50 casos pasan, y al menos 3 de ellos fallan de verdad si comentás el escapado de `wrapUntrustedContent` (probalo y dejalo escrito — un test que no puede fallar no prueba nada).
- CI verde en real. Actualiza `../Jin_Docs/STATUS.md`.

RESTRICCIONES:
- Ningún payload del corpus puede ejecutarse contra APIs reales (Gmail/Canvas/Modal) — todo mockeado.

USA CLAUDE OPUS 4.8 (auditoría de seguridad).

ANTES DE IMPLEMENTAR: Propón el plan. Espera aprobación.
```

---

## Fase 9 — Funciones del blueprint nunca agendadas (post-producción)

> Mismo origen (auditoría de cobertura 2026-08-05): están en `BLUEPRINT.md` con spec propia pero ninguna fase 1-7 las cubría. Ninguna bloquea el despliegue — van después de 7.3, en el orden que el owner prefiera.

### 9.1 [ANTIGRAVITY] — Notion + comando `/audio` (repo: Jin_Core)

```
Notion es un objetivo declarado del proyecto (BLUEPRINT §1.1: "productividad: Gmail, Calendar, Notion"), tiene spec propia (§7.4), aparece como destino del comando `/audio` del bot (§8.2) y como ejemplo de HITL `notify` (§9.1). Verificado: cero código de Notion en el repo, y `/audio` no está entre los comandos implementados del bot.

ANTES DE ESCRIBIR CÓDIGO:
1. Lee `../Jin_Docs/docs/BLUEPRINT.md` §7.4, §8.2, §9.1 y `AGENTS.md` §5.1.
2. Copiá el patrón ya establecido por tus integraciones anteriores: `src/integrations/google/` y `src/integrations/canvas/` (cliente REST + tools registradas en `src/tools/registry.ts` + `toolExecutorRegistry.register()` en el `onModuleInit()` del módulo).

TU TAREA:
- `src/integrations/notion/`: cliente con integration token internal (scoped a un workspace), vía `env.schema.ts` fail-fast como toda credencial.
- Tools nuevas en `registry.ts` con su `hitlLevel` explícito. Referencia del blueprint: "añadir tarea a Notion" es `notify` (§9.1) y "comentario en Notion" es reversible a efectos de timeout (§9.4). Usos declarados en §7.4: logs de entrenamiento físico, resúmenes académicos, base de enlaces.
- Comando `/audio` en el bot: recibe un mensaje de voz de Telegram, lo transcribe y guarda el resultado en Notion. La transcripción necesita un proveedor — proponé cuál en el plan (OpenAI ya está como dependencia por los embeddings de memoria) y justificá el costo contra el budget guard.
- Todo contenido que vuelva de Notion es input externo: `wrapUntrustedContent` obligatorio antes de que toque el contexto del LLM.

CRITERIOS DE ÉXITO:
- Un audio enviado por Telegram termina como página/bloque real en Notion, con su entrada en el audit log.
- Las tools nuevas están en el contrato OpenAPI regenerado y tienen tests de la matriz tool × hitlLevel (BLUEPRINT §13.1).
- CI verde en real. Actualiza `../Jin_Docs/STATUS.md`.

ANTES DE IMPLEMENTAR: Propón el plan. Espera aprobación.
```

### 9.2 [CLAUDE CODE] — GitHub App + capacidad git real (repos: Jin_Executor + Jin_Core)

```
BLUEPRINT §7.3 especifica GitHub App (no PAT) con tokens de instalación efímeros (1 h) generados on-demand por el Executor e inyectados al pod sin persistir. Verificado: cero código de GitHub en ambos repos. La consecuencia real está escrita en el propio código: `mergeAgentBranch` (registry.ts) existe como guardrail pero lanza 501, con el comentario "no existe hoy ninguna tool que le dé a un sub-agente la capacidad de producir una branch real" — y `startPreviewService` recibe los archivos inline (`files`) en vez de clonar una branch, documentado como extensión futura en ADR 0006.

O sea: esta fase cierra el ciclo de "agentes que producen código de verdad", que hoy está cortado.

ANTES DE ESCRIBIR CÓDIGO:
1. Lee `../Jin_Docs/docs/BLUEPRINT.md` §7.3, §9.1 (ejemplos HITL: `git commit` local = notify, `git push` = confirm, `git push --force` = dual-confirm) y ADR 0005 punto 9 + ADR 0006.
2. Lee `Jin_Executor/src/` completo — el token va inyectado al pod por el Executor, jamás por core (regla arquitectónica dura de §4.1: core no habla con K8s).

TU TAREA:
- GitHub App con permisos scoped: generación de installation token efímero en el Executor, on-demand, nunca persistido ni logueado.
- Tools git en `registry.ts` con los niveles exactos del blueprint §9.1: `git commit` local → `notify`, `git push` → `confirm`, `git push --force` → `dual-confirm`. Ninguna tool git puede ser `auto`.
- Implementar `mergeAgentBranch` de verdad (quitar el 501) ahora que la capacidad existe.
- Egress: los pods que hagan git ops necesitan `github.com` en su whitelist — y solo eso.

CRITERIOS DE ÉXITO:
- Un sub-agente puede producir una branch real y `mergeAgentBranch` la mergea tras aprobación HITL.
- El token nunca aparece en logs, ni en el audit log, ni persiste tras la vida del pod (test que lo pruebe).
- CI verde en real. Actualiza `../Jin_Docs/STATUS.md`.

RESTRICCIONES:
- GitHub App, no PAT (§7.3 es explícito). Sin `git push --force` a `main` bajo ninguna circunstancia, ni siquiera con dual-confirm.

USA CLAUDE OPUS 4.8 (credenciales efímeras + RBAC = frontera de seguridad).

ANTES DE IMPLEMENTAR: Propón el plan. Espera aprobación.
```

### 9.3 [CLAUDE CODE] — RAG de corpus propio en pgvector (repo: Jin_Core)

```
BLUEPRINT §3.3.1 traza una frontera explícita que hoy solo está construida a la mitad:
- "pgvector = corpus. Correos indexados, PDFs de Canvas, notas. Grande, con JOINs relacionales." → NO EXISTE.
- "sqlite-vec = memoria del agente. Hechos, preferencias, episodios." → construido en Fase 4.3.

Verificado: los embeddings del repo viven solo en `src/memory/` (sqlite-vec, `text-embedding-3-large`). La imagen de Postgres ya trae pgvector (`pgvector/pgvector:0.8.5-pg16`, tanto en producción como en `docker-compose.dev.yaml`) pero ningún módulo la usa. §6.4 lo confirma: "Corpus propio (correos indexados, notas, tareas): embeddings guardados en pgvector".

ANTES DE ESCRIBIR CÓDIGO:
1. Lee `../Jin_Docs/docs/BLUEPRINT.md` §3.3, §3.3.1 y §6.4 completas.
2. Lee `src/memory/` entero — reusá `EmbeddingProvider` (ya existe, ya elige modelo y guarda `modelo_embedding` por entrada). NO escribas un segundo proveedor de embeddings.
3. `AGENTS.md` §5.1 punto 2: todo lo que se indexa pasa por `sanitizeForIndexing()` (ya existe en `injection-sanitizer.ts`, hoy sin consumidor real — este es su caller).

TU TAREA:
- Schema Drizzle + migración para el corpus con índice HNSW (§3.3; IVFFlat como fallback documentado si la RAM aprieta).
- Pipeline de indexación de al menos una fuente real de las tres declaradas (correos, PDFs de Canvas, notas) — elegí la de mayor valor y dejá las otras documentadas como extensión, sin stubs vacíos.
- Búsqueda con JOIN relacional real (ese es el argumento entero de §3.3 para elegir pgvector sobre Qdrant: "un JOIN entre tasks y task_embeddings es SQL nativo").
- Métrica `rag_hit_ratio` (§10.1 la lista y no existe).

CRITERIOS DE ÉXITO:
- Una consulta del agente recupera contexto real del corpus, con su JOIN, y el contenido recuperado llega envuelto (`wrapUntrustedContent`) al prompt.
- Tests de integración con Postgres real (testcontainers, ya configurado).
- CI verde en real. Actualiza `../Jin_Docs/STATUS.md`.

RESTRICCIONES:
- No mezcles corpus con memoria: son dos almacenes con dos ciclos de vida distintos, la frontera de §3.3.1 no se negocia.

ANTES DE IMPLEMENTAR: Propón el plan. Espera aprobación.
```

### 9.4 [ANTIGRAVITY] — Alerta matutina 06:00 (repo: Jin_Core)

```
BLUEPRINT §7.1, última línea del Shadowing Académico: "Genera bloques de estudio en Google Calendar (con HITL notify). **Alerta matutina 06:00 con resumen de prioridades**". Verificado: el cron nocturno de las 00:00 existe y funciona (`src/integrations/canvas/shadowing.service.ts`, tuyo desde Fase 3.1), pero no hay ningún cron a las 06:00 ni nada que arme un "resumen de prioridades".

Es la mitad que falta del ciclo: hoy el sistema analiza a medianoche y no te dice nada hasta que vos preguntás.

ANTES DE ESCRIBIR CÓDIGO:
1. Lee `../Jin_Docs/docs/BLUEPRINT.md` §7.1 y §10.4.
2. Lee tu propio `shadowing.service.ts` — el cron de 00:00 ya deja el análisis hecho; esta fase decide qué se persiste de esa corrida para poder resumirla 6 h después (hoy no se guarda nada reutilizable, verificalo antes de asumir).

TU TAREA:
- Cron 06:00 local que envía por Telegram el resumen de prioridades del día.
- Decidí y documentá de dónde salen las prioridades: reusar el resultado de la corrida de las 00:00 (requiere persistirlo) o recalcular. Si recalculás, justificá el costo contra el budget guard — son dos llamadas a LLM por día en vez de una.
- El resumen es informativo: `auto` en términos de HITL, sin aprobación. Pero pasa por el audit log igual.

CRITERIOS DE ÉXITO:
- A las 06:00 llega un mensaje de Telegram con las prioridades reales del día, no un placeholder.
- Si la corrida de las 00:00 falló, el mensaje de las 06:00 lo dice explícitamente en vez de mentir con un resumen vacío.
- CI verde en real. Actualiza `../Jin_Docs/STATUS.md`.

ANTES DE IMPLEMENTAR: Propón el plan. Espera aprobación.
```

### 9.5 [CLAUDE CODE] — Feature flags en caliente (repo: Jin_Core + Jin_Infra)

```
BLUEPRINT §12.3: "ConfigMap `feature-flags.yaml`. Reload en caliente vía SIGHUP en core. Uso: activar/desactivar integraciones, cambiar HITL levels de tools individuales (con dual-confirm), rutear entre modelos". Verificado: cero código de feature flags, cero ConfigMap.

OJO CON LA TENSIÓN DE DISEÑO, resolvela explícitamente en el plan: §9.3 dice "Prohibido: el LLM no puede cambiar el hitlLevel de una tool en runtime" y §12.3 dice que un humano SÍ puede vía flag con dual-confirm. No se contradicen (uno es el LLM, otro es el owner) pero la implementación tiene que hacer imposible el primero mientras habilita el segundo — y "cambiar el HITL level de una tool" ya está declarado como acción `dual-confirm` en la tabla de §9.1.

ANTES DE ESCRIBIR CÓDIGO:
1. Lee `../Jin_Docs/docs/BLUEPRINT.md` §12.3, §9.1 y §9.3.
2. Lee `src/tools/registry.ts` — hoy los `hitlLevel` son estáticos y `Object.freeze`-ados a propósito. Ese es el invariante que estás tocando: pensalo dos veces y documentá la decisión.

TU TAREA:
- ConfigMap + carga en caliente (SIGHUP) sin redeploy.
- Flags de: activar/desactivar integraciones, override de `hitlLevel` por tool, ruteo entre modelos.
- Todo cambio de flag que toque un `hitlLevel` queda en el audit log con su aprobación dual-confirm. Un override jamás puede BAJAR un nivel sin dual-confirm; subirlo (más restrictivo) puede ser `confirm`.
- Tests: el LLM no puede alterar un flag por ninguna vía (ni tool, ni prompt injection — cruzá esto con el golden set de 8.2).

CRITERIOS DE ÉXITO:
- Cambiar un flag y recargar sin reiniciar el pod, verificado.
- Test que prueba que bajar un `hitlLevel` sin dual-confirm es imposible.
- CI verde en real. Actualiza `../Jin_Docs/STATUS.md`.

USA CLAUDE OPUS 4.8 (toca el invariante central del HITL).

ANTES DE IMPLEMENTAR: Propón el plan. Espera aprobación.
```

### 9.6 [ANTIGRAVITY] — Comandos de administración en la CLI (repo: Jin_CLI)

```
BLUEPRINT §8.3 declara para la CLI: "Ink + **Inquirer.js para menús navegables con teclas de flecha**" y "casos de uso: desarrollo local, debugging, **admin tasks (rotar tokens, forzar backup, ver logs recientes)**". La Fase 6.4 construyó 7 comandos (login/status/tasks/approve/reject/chat/memory) — todos de operación, ninguno de administración, y sin menús navegables.

ANTES DE ESCRIBIR CÓDIGO:
1. Lee `../Jin_Docs/docs/BLUEPRINT.md` §8.3.
2. Lee tu propio `Jin_CLI/source/` de la Fase 6.4 — el cliente tipado y el auth storage ya están, reusalos.
3. Verificá qué endpoints existen realmente en `Jin_Core/contracts/openapi.json` antes de diseñar un comando: "forzar backup" y "ver logs recientes" pueden no tener endpoint todavía. Si falta, decilo en el plan en vez de inventar la ruta — se coordina con Claude Code en STATUS.md.

TU TAREA:
- Menús navegables con teclas de flecha donde aporten (la bandeja de aprobaciones es el caso obvio: navegar y aprobar sin copiar UUIDs a mano).
- Comandos de admin de §8.3, solo los que tengan endpoint real detrás.
- Cero tipos copiados a mano (regla de oro #11).

CRITERIOS DE ÉXITO:
- Aprobar una acción HITL navegando con flechas, sin pegar un requestId.
- CI verde en real. Actualiza `../Jin_Docs/STATUS.md`.

ANTES DE IMPLEMENTAR: Propón el plan. Espera aprobación.
```

---

## Plantilla genérica

Cuando abras una nueva sesión no cubierta arriba, usa esta plantilla:

```
[NOMBRE DE LA TAREA] (repo: Jin_XXX)

ANTES DE ESCRIBIR CÓDIGO:
1. Lee `../Jin_Docs/docs/BLUEPRINT.md` sección [X].
2. Lee `../Jin_Docs/AGENTS.md` completo.
3. Lee [otros docs relevantes].
4. Revisa `../Jin_Docs/STATUS.md` y verifica que no hay colisión con el otro IDE (solo posible en Jin_Core).

TU TAREA:
[Descripción específica con archivos exactos]

CRITERIOS DE ÉXITO:
- [Lista concreta y verificable]

RESTRICCIONES:
- [Lista de "no hacer"]

USA [MODELO] para esta tarea porque [razón].

ANTES DE IMPLEMENTAR:
Propón el plan detallado. Espera aprobación explícita antes de escribir código.
Actualiza `../Jin_Docs/STATUS.md` al empezar y al terminar.
```

---

## Reglas para todos los prompts

1. **Siempre indicar el repo** donde se trabaja. Sin repo declarado, el LLM no escribe código.
2. **Siempre pedir el plan primero.** Nunca dejes que el LLM escriba código en el primer turno.
3. **Siempre citar los docs a leer.** No asumas que el LLM recuerda.
4. **Siempre especificar el modelo** si es un caso donde la elección importa.
5. **Siempre pedir actualización de `STATUS.md`** al inicio y al fin.
6. **Siempre incluir criterios de éxito verificables**, no aspiraciones vagas.
7. **Siempre listar restricciones explícitas**, no confíes en que el LLM las infiera de AGENTS.md.
