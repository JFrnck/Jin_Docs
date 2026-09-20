# ADR 0010 — Modos de autonomía del HITL y reclamo atómico de aprobaciones

## Contexto

Dos cambios sobre el mismo núcleo de seguridad (`Jin_Core/src/hitl/`), decididos el 2026-09-19 con el owner (plan aprobado línea por línea, `CLAUDE.md` §2.1):

1. **Issue Jin_Core #36 — carrera de doble aprobación.** `ApprovalExecutionService.resolveAndExecute()` leía el pendiente, ejecutaba la acción real y recién después lo borraba, sin reclamarlo antes. **Reproducido con Postgres real: 10 aprobaciones concurrentes ejecutaron `sendEmail` 10 veces.** Escenarios reales: aprobar desde Web y Telegram a la vez, doble clic, reintento de red; o una falla del audit/borrado *después* de ejecutar, que dejaba el pendiente vivo y re-ejecutable; o el barrido horario de `TimeoutService` compitiendo con la ejecución.
2. **Modos de autonomía.** El owner quiere HITL completo por defecto, pero poder bajar la fricción cuando trabaja con el agente: un modo **semiautomático** (solo pide aprobación para git/merges, correos y borrar eventos futuros) y uno **automático**. Esto choca *a propósito* con la regla de oro #4 ("el HITL level de una tool jamás lo decide el LLM; es estático y solo cambiable con dual-confirm humano") — el diseño tiene que **cumplirla, no esquivarla**.

## Decisiones

### 1. Reclamo atómico de la ejecución (issue #36)

`pending_approvals.executing_at` se setea con `UPDATE … WHERE executing_at IS NULL RETURNING` (`DualConfirmService.claimForExecution`): en Postgres solo una llamada gana, aunque lleguen N a la vez. Nuevo orden de `resolveAndExecute`: aprobación → **claim** → **audit de la intención** → ejecutar → borrar.

- **A lo sumo una vez**, no al menos una: para una acción irreversible ante un tercero (`sendEmail`), ejecutar de menos es recuperable (el owner reaprueba); ejecutar de más no.
- **Fail-closed**: la intención (`approved`) se audita *antes* de ejecutar; si la cadena está bloqueada o la DB falla, la acción no ocurre y el claim se libera.
- **Sin reintento automático** (regla de oro #9, misma lógica que el timeout que nunca aprueba): si el executor falla, el pendiente vuelve a estar disponible con `execution_error` visible en la API y exige una **nueva aprobación humana**. El fallo queda auditado (`execution_failed`).
- **Reclamo trabado** (proceso muerto entre el claim y el final; >15 min): la acción *pudo o no* haberse ejecutado, así que **nunca se reintenta ni se descarta sola**. `TimeoutService` emite `hitl.approval.stuck` (alerta Telegram horaria) y el owner lo limpia con `/reject` tras comprobarlo.
- `TimeoutService` usa el **mismo primitivo** para descartar/abandonar: una expiración nunca compite con una ejecución.
- La primera aprobación de un `dual-confirm` es condicional: dos "primeras" simultáneas no cuentan como "segunda" saltándose los 30 s.
- Rechazar no puede llegar a mitad de una ejecución.

### 2. Los modos son estado del owner, no una tool

Tres modos: `supervised` (default), `semi-auto`, `auto`. Una tabla singleton (`autonomy_mode_state`, migración 0011, CHECK de modo y de `id = 1`) **sembrada en `supervised`**.

- **Solo `confirm` se relaja, y solo a `notify`** (se ejecuta y *avisa*). Nunca: `dual-confirm` (el **piso**, decisión del owner, reglas #4 y #7), tools `humanDecision` (`resolveAgentConflict`: es una decisión, no una acción — automatizarla la vaciaría de sentido), ni lo ya laxo.
- `semi-auto` conserva `confirm` en las tools marcadas **`guardedInSemiAuto`** en el registry: `sendEmail`, `deleteCalendarEventFuture`, `mergeAgentBranch` (+ las tools git de la Fase 9.2). Las marcas son estáticas e `Object.freeze`-adas como el resto del registry.
- **Bajar la protección exige `dual-confirm` real** (2 aprobaciones ≥30 s); **subirla es inmediata**. **Renovar** un modo relajado cuenta como bajar (extiende el tiempo sin protección).
- **Caducan solos** y vuelven a `supervised` (auto 4 h, semi 24 h por defecto; máx. 24 h / 72 h; `config/autonomy.yaml`), por lectura perezosa + cron por minuto. Un modo relajado *sin* caducidad se lee como `supervised`.
- **Freno de emergencia**: más de N acciones autoejecutadas por hora (20) ⇒ vuelve a `supervised` y avisa. Contador atómico en la propia fila.
- **Ante cualquier duda, `supervised`**: fila ausente, modo inválido, sin caducidad, caducado.
- **El LLM no tiene camino hacia el interruptor.** El cambio de modo reutiliza el patrón de `FeatureFlagsService` (Fase 9.5): un "tool" virtual `autonomyModeChange` registrado en `ToolExecutorRegistry` pero **no** en `registry.ts`; el agente solo expone lo que está en el registry y `classifyToolCall` lanza `UnknownToolError` para todo lo demás. Los únicos caminos que lo escriben: `POST /api/autonomy` (JWT), Telegram `/mode` (`ownerChatId`) y el ejecutor de un dual-confirm aprobado.

### 3. Una sola puerta de decisión: `HitlPolicyService`

Había **4 sitios** que clasificaban el nivel (`AgentService`, `OrchestratorService`, y los dos `Google*ToolsService`) y solo `AgentService` consultaba los overrides de la Fase 9.5. Un modo global habría sido imposible de garantizar. `HitlPolicyService.decide()` compone, en orden: nivel **estático** del registry (`classifyToolCall`, única fuente del nivel base) → overrides de feature flags → modo de autonomía. Cada capa solo ajusta lo que dejó la anterior.

**Un override explícito del owner (flag) gana sobre el modo**: si endureció una tool, un interruptor general no lo deshace. Los `Google*ToolsService` no relajan (solo pueden ser más estrictos).

### 4. `notify` ahora notifica de verdad

Hallazgo durante el diseño: en el camino del agente, `notify` solo escribía una fila `notified` en el audit — **ningún listener avisaba al owner** (BLUEPRINT §9.1 promete "notificación post-hoc"). Con modos que convierten `confirm` en `notify` eso habría sido ejecutar acciones sin ninguna señal. `AgentService` emite `hitl.action.notified`; Telegram avisa, y si un modo la relajó lo dice explícitamente (*"se ejecutó SIN pedirte aprobación (autonomy:auto)"*). El audit registra el motivo.

## Consecuencias

- En **modo automático, `sendEmail` se envía sin aprobación** (el piso es `dual-confirm`). Aceptado explícitamente por el owner; se mitiga con caducidad corta, freno por hora, aviso post-hoc con motivo y audit de cada acción. Un correo hostil que logre inyectar una instrucción podría exfiltrar por correo mientras el modo automático esté activo: por eso el default es `supervised`, dura horas, y `semi-auto` mantiene `sendEmail` bajo aprobación.
- Hoy **ninguna tool del registry es `dual-confirm`**; el piso se prueba con decisiones sintéticas y quedará ejercido de verdad cuando existan (git force-push, Fase 9.2).
- Requiere dos migraciones (0010 y 0011, ambas retrocompatibles) que corre el Job de `Jin_Infra`.
- El dashboard y la CLI necesitan un interruptor (PR de Web/CLI) y mostrar `executionError`.

## Alternativas consideradas

- **Modo como flag en `feature-flags.yaml`:** rechazado — se cambia por PR + SIGHUP, sin caducidad ni notificación, y no es un cambio operativo de minutos como pide el uso real.
- **Dejar `dual-confirm` automatizable en modo automático:** rechazado por el owner (regla de oro #7); exigiría un ADR que la modifique.
- **Reintento automático tras fallo de ejecución:** rechazado — reintentar una acción irreversible sin humano es la clase de comportamiento que la regla #9 prohíbe.
- **Relajar por tool en vez de por modo global:** ya existe (`hitlOverrides`, Fase 9.5) y exige dual-confirm por cada tool; los modos resuelven el caso "estoy trabajando con el agente, dejalo fluir".
- **Un solo `confirm` para activar un modo:** rechazado — un toque impulsivo, o un móvil desbloqueado, bastaría para quitar la protección.
