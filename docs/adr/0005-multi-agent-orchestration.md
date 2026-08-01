# ADR 0005 — Orquestación multi-agente: ledger persistido + escalera de decisión

## Contexto

BLUEPRINT §6/§9 y la Fase 5.1 establecen un agente único por turno. El owner pidió (2026-07-25) que un objetivo pueda descomponerse en tickets delegados a sub-agentes en paralelo, coordinados sin comunicación directa entre ellos. Decisión ya tomada por el owner, no rediscutida acá: coordinación vía task ledger estilo Jira persistido en Postgres, mediado por el orquestador, con escalera de decisión (bajo riesgo = auto, material = HITL al owner).

## Decisiones

### 1. Sub-agente = instancia adicional de `AgentService`, no una clase nueva

Evita duplicar tool-calling/plan-and-solve/self-correction/sanitización ya construidos y probados en Fase 5.1. El orquestador es la única pieza nueva de verdad.

### 2. Ledger persistido en 3 tablas Postgres (`agent_orchestration_runs`/`agent_tickets`/`agent_ticket_comments`)

Mismo criterio que `pending_approvals`: un ticket puede quedar bloqueado días esperando HITL y debe sobrevivir restarts. El hilo de comentarios (no solo estado) es lo que hace el conflicto "visible" en vez de suprimido, tal como exige el requisito del owner.

### 3. Paralelismo = concurrencia async por lotes de dependencias, con tope configurable

Node es single-threaded; "paralelo" es I/O concurrente (llamadas a modelo/tools), no cómputo real en paralelo — correcto para esta carga de trabajo. Los tickets se agrupan en lotes según qué dependencias ya están `done`; dentro de un lote corren con `Promise.all` en chunks de tamaño `max_concurrent_sub_agents` (config, default 3, mismo mecanismo de guarda dura que `max_iterations_per_turn`/`max_consecutive_tool_failures` de 5.1).

### 4. Visibilidad del board es causal, no instantánea entre pares del mismo lote

Un sub-agente ve el estado del board (tickets ya resueltos de lotes anteriores) al arrancar, inyectado como par sintético `user`/`assistant` en su `history` — mismo mecanismo que Fase 5.3 usa para memoria recuperada. Tickets que corren en el mismo lote no se ven entre sí en tiempo real (limitación real del modelo de ejecución, no un bug oculto); las contradicciones entre ellos se detectan y resuelven en la pasada de reconciliación final, no a mitad de vuelo.

### 5. Detección y resolución de conflictos: una sola pasada de reconciliación, TaskProfile `reasoning_heavy`

El orquestador no implementa un detector de contradicciones ad-hoc: al cerrar el run, un LLM (Opus 4.8 vía `reasoning_heavy`) recibe el board completo (tickets + resultados + comentarios) y devuelve JSON estructurado con conflictos clasificados `low`/`material`. Bajo riesgo se resuelve en modo-auto (comentario `resolution` con el porqué, tal como pide el requisito del owner de "dejar registrada la resolución"). Material dispara la tool sintética `resolveAgentConflict` (`hitlLevel: 'confirm'`), reusando el mecanismo de aprobación de Telegram existente — el owner sigue siendo el nivel máximo de la escalera sin inventar un canal nuevo.

### 6. Presupuesto: `sessionId` derivado (`{runId}:{ticketId}`) por sub-agente, topes globales intactos

Cada sub-agente obtiene su propio techo de sesión (nadie hace runaway solo) pero los topes diario/horario y el kill switch (`budget_daily_usage`/`budget_hourly_usage`/`budget_kill_switch`) son globales por diseño desde Fase 4.1 y no distinguen `sessionId` — fan-out de sub-agentes no es una vía para evadirlos. `BudgetService` gana `getSessionUsage()` público (antes privado) para que el orquestador reporte el presupuesto agregado del turno como la suma real de sus sub-agentes.

### 7. Kill switch corta el run completo, no solo la llamada en curso

Antes de lanzar cada lote nuevo el orquestador chequea `KillSwitchService.isActive()`. Si se activa a mitad de un lote en ejecución, la próxima llamada de cualquier sub-agente a `BudgetGuardedModelRouter.complete()` ya lanza `KillSwitchActiveError` (mecanismo existente desde 4.1, sin cambios) — el orquestador la distingue explícitamente de un fallo de ticket normal y marca el run entero `killed` sin lanzar más lotes.

### 8. Profundidad de delegación = 1, aplicada estructuralmente

Un sub-agente nunca recibe la tool de orquestación en su `allowedTools` (el orquestador construye ese subset y jamás se incluye a sí mismo) — no hace falta un contador de profundidad en runtime porque la restricción es estructural: no existe ningún camino por el que `AgentService.runTurn()` pueda invocar `OrchestratorService`.

### 9. `mergeAgentBranch` declarada pero no implementada — 501 documentado

PROMPTS.md exige que el trabajo de código de un sub-agente viva en `feature/agent/<ticket>` y el merge a main sea una tool `confirm`. Hoy ninguna tool registrada le da a un agente la capacidad de escribir código o crear una branch real — Jin_Executor ejecuta pods sandboxed sin acceso a git. Se declara la tool (`hitlLevel: 'confirm'`) para dejar el guardrail listo, con executor que lanza `AgentBranchMergeNotImplementedError` (501) — mismo patrón que `CalendarNotImplementedError` (Fase 3.1) y el stub de `ModalService` (Fase 2.3): falla ruidoso, no una implementación parcial silenciosa. Se activa cuando exista una tool real de edición de código.

### 10. Atribución sin migrar `audit_log`/`pending_approvals`

`actor` (audit_log) y `planSummary` (pending_approvals) ya son texto libre. `AgentTurnInput` gana `actorLabel?: string` (default `'agent'`, retrocompatible); el orquestador pasa `agent:{ticketId}`. Sin columnas nuevas, sin romper ningún caller existente (Telegram/tests que no pasan `actorLabel` se comportan exactamente igual que hoy).

## Consecuencias

- El agent loop de Fase 5.1 queda prácticamente intacto: dos campos opcionales nuevos en `AgentTurnInput`, cero cambios de comportamiento para callers existentes.
- El ledger es la primera pieza del sistema pensada explícitamente para que el dashboard de Fase 6 la renderice como board — decisión de diseño ya anticipada, no retrofit.
- Deuda documentada, no silenciosa: `mergeAgentBranch` es un 501 hasta que exista una tool de edición de código real.
- Visibilidad del board entre tickets del mismo lote es causal, no instantánea — aceptado como limitación real del modelo async, resuelto en la pasada de reconciliación final en vez de fingir tiempo real que no existe.
- `AGENT_CONFIG` pasó a ser exportado desde `AgentModule` (antes solo `AgentService`) — lo necesita `OrchestratorModule` para `max_concurrent_sub_agents`, sin duplicar la carga de `config/agent.yaml` en una segunda factory.

## Alternativas consideradas

- **Comunicación peer-to-peer entre sub-agentes:** descartada explícitamente por el owner — pierde trazabilidad y multiplica riesgo runaway (ver PROMPTS.md §5.4).
- **Detector de conflictos incremental (a mitad de cada lote) en vez de una pasada final:** más "tiempo real", pero requiere que cada sub-agente pause a mitad de su propio turno a esperar a sus pares del mismo lote — contradice el modelo de "un turno de `AgentService` es una unidad atómica" de Fase 5.1 y complica sustancialmente la máquina de estados sin necesidad clara (PROMPTS.md solo exige que la reconciliación exista, no que sea incremental).
- **`sessionId` compartido entre todos los sub-agentes de un run (en vez de derivado por ticket):** simplificaría la suma de presupuesto, pero le quitaría a cada sub-agente su propio techo individual — un solo ticket runaway agotaría el presupuesto de sesión de TODO el run, exactamente lo que el cap de sesión existe para evitar.
- **Librería de concurrencia (`p-limit`/`p-queue`) para el chunking:** innecesaria — el caso de uso (batches acotados por dependencias, tope bajo configurable) se resuelve con un chunker de <15 líneas sin dependencia nueva.
