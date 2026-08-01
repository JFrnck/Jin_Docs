# ADR 0002 — `request_id` en `audit_log` + tabla mutable `pending_approvals`

## Contexto

BLUEPRINT §9.5 (versión original) define `audit_log` con `approval_status` tomando valores `'pending' | 'approved' | 'rejected' | 'timeout' | 'abandoned'`, lo que sugiere que una fila "pendiente" evoluciona hasta su resolución. Pero el mismo blueprint (regla de oro #12) y AGENTS.md §5.6 exigen que el log sea insert-only con hash chain: **"modificar una row histórica es un incidente de seguridad"**. Actualizar en el sitio el `approval_status` de una fila ya escrita invalidaría su `current_hash` y el de toda fila posterior en la cadena — es una contradicción directa con el propio diseño.

Además, el schema original no tiene ninguna columna que correlacione la fila inicial de una acción (`tool_call`, `pending`) con la fila que registra su desenlace (`approval`/`rejection`/`timeout`). Sin esa correlación, no hay forma de reconstruir "¿qué pasó con esta aprobación específica?" a partir del log.

Detectado durante el diseño de la Fase 2.2 (HITL classifier + audit log), antes de escribir código, siguiendo AGENTS.md §10.4 ("si detectas que el BLUEPRINT está incorrecto: detente, explica, propón, espera aprobación").

## Decisión

1. **`audit_log` gana una columna `request_id UUID NOT NULL`.** Todas las filas que describen el mismo evento lógico (la petición inicial y su eventual resolución) comparten el mismo `request_id`, generado al momento de clasificar la tool call. El log sigue siendo 100% insert-only: cada transición de estado es una fila **nueva**, nunca un UPDATE.

2. **El estado "en espera de aprobación ahora mismo" se separa en una tabla nueva y mutable, `pending_approvals`** (PK = `request_id`), distinta del log inmutable:
   - Se inserta cuando `classifyToolCall` determina `confirm` o `dual-confirm`.
   - Se actualiza in-place mientras la aprobación está en curso (ej. registrar la primera de las dos confirmaciones del `dual-confirm`, con su timestamp, para poder exigir los ≥30s antes de aceptar la segunda).
   - Se borra (o marca resuelta) cuando `audit.service.ts` escribe la fila terminal correspondiente en `audit_log`.
   - Persistida en Postgres, no en memoria: con 1 réplica y rolling updates (BLUEPRINT §12.2), una aprobación pendiente en memoria se perdería en cada restart sin que el timeout la hubiera resuelto explícitamente — violaría en la práctica la regla "el timeout nunca aprueba automáticamente" al convertir un restart en una pérdida silenciosa de estado.

## Consecuencias

- El hash chain de `audit_log` queda genuinamente inmutable — ninguna fila se toca después de escrita.
- Reconstruir el ciclo de vida de una aprobación es un filtro por `request_id`, no una inferencia por proximidad de timestamp/tool_name (frágil y ambigua).
- Se añade una tabla más al schema de Jin_Core; su ciclo de vida (insert al pedir, update mientras se decide, delete/mark-resolved al terminar) es explícitamente distinto del de `audit_log` — no deben confundirse ni fusionarse.

## Alternativas consideradas

- **UPDATE de la fila original + recalcular el hash chain desde ese punto:** técnicamente posible pero contradice la premisa de "hash chain sagrado"; cualquier proceso que pueda recalcular hashes retroactivamente es, en la práctica, un proceso que puede falsificar el historial. Rechazada.
- **Correlacionar por `inputs_hash` + ventana de tiempo:** frágil (dos llamadas idénticas a la misma tool en poco tiempo colisionarían) e implícito. Rechazada.
- **Guardar el estado pendiente dentro de `audit_log` con una columna adicional de "fila vigente":** mezclaría datos mutables dentro de una tabla cuya única invariante es ser inmutable — más fácil de violar por accidente que tener dos tablas con contratos distintos y claros.
