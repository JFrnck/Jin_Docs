# ADR 0001 — Cuatro niveles de HITL, estáticos por tool

## Contexto

Jin orquesta agentes que ejecutan acciones con distinto grado de reversibilidad e impacto: desde leer un correo hasta hacer `git push --force`. Se necesita un mecanismo de control humano (Human-in-the-Loop) que sea:

- Proporcional al riesgo real de cada acción (no todo necesita aprobación, pero nada destructivo se ejecuta sin ella).
- A prueba de manipulación: un prompt injection o un bug del LLM no debe poder degradar el nivel de control de una tool.
- Simple de razonar y de auditar (BLUEPRINT §15, regla de oro #7: "cuando dudes entre `confirm` y `dual-confirm`, elige `dual-confirm`").

## Decisión

Cuatro niveles, declarados de forma **estática** por cada tool en `src/tools/registry.ts`, nunca decididos en runtime por el LLM ni derivados de los `inputs` de la llamada:

| Nivel | Comportamiento | Ejemplo |
| --- | --- | --- |
| `auto` | Ejecuta sin notificar | Leer correos, buscar en Calendar |
| `notify` | Ejecuta y notifica después | Crear evento, `git commit` local |
| `confirm` | Espera 1 aprobación | `git push`, responder correo |
| `dual-confirm` | 2 aprobaciones separadas por ≥30s | Borrar archivos, revocar tokens |

Por qué 4 y no 2 o 3: la línea "destructivo o no" es insuficiente — hay acciones irreversibles pero de bajo riesgo real (crear un evento de Calendar) que no ameritan fricción, y acciones reversibles pero catastróficas en potencia (`git push --force`) que sí. Cuatro niveles separan: **ejecutar sin fricción** (`auto`), **ejecutar con trazabilidad post-hoc** (`notify`), **pausar y pedir una decisión** (`confirm`), y **pausar dos veces con separación temporal forzada** (`dual-confirm`, contra el "apruebo todo por reflejo" en el móvil — BLUEPRINT §9.2).

`classifyToolCall(toolName, inputs)` acepta `inputs` en su firma (para computar el hash de auditoría), pero **el nivel se determina únicamente por `toolName`** contra el registry. Un tool no registrado lanza `UnknownToolError` — nunca se asume `auto` por defecto (fail-safe explícito, AGENTS.md 1.4).

Cambiar el `hitlLevel` de una tool existente requiere PR con revisión humana + `dual-confirm` para que el cambio tome efecto en producción (BLUEPRINT §15, regla #4).

## Consecuencias

- El clasificador es una función pura y 100% testeable (matriz tool × nivel, AGENTS.md 6.1).
- Añadir una tool nueva siempre implica una decisión humana explícita de su nivel — no hay default implícito.
- El campo `inputs` en la firma queda sin usar para clasificación hoy; se documenta explícitamente en el código para no confundir a un futuro lector (por qué existe si no se usa).

## Alternativas consideradas

- **2 niveles (auto/confirm):** insuficiente granularidad — mezclaría "crear evento de Calendar" con "hacer push a producción" bajo la misma fricción.
- **Nivel decidido dinámicamente por el LLM según el contexto:** rechazado de plano — es exactamente el vector que un prompt injection explotaría para escalar privilegios. Es la regla de oro #4 del blueprint.
