# ADR 0011 — Alerta matutina 06:00 (9.4) y comandos de administración de la CLI (9.6)

## Contexto

Dos funciones del BLUEPRINT que ninguna fase 1-7 cubría (auditoría de cobertura 2026-08-05), aprobadas por el owner el 2026-09-19:

- **§7.1**: "Alerta matutina 06:00 con resumen de prioridades" — el cron de las 00:00 analizaba Canvas pero devolvía el resumen sin guardarlo, y nadie avisaba nada hasta que el owner preguntaba.
- **§8.3**: la CLI debía ser "Ink + Inquirer.js para menús navegables" con "admin tasks"; la Fase 6.4 solo construyó comandos de operación, sin menús, y aprobar exigía copiar el UUID a mano.

## Decisiones

### 9.4 — Reutilizar la corrida de las 00:00, no recalcular
Cada corrida se persiste en `shadowing_runs` (migración 0012), **incluida la fallida** (`status = 'failed'` + `error`). A las 06:00 `MorningAlertService` lee la última corrida de las últimas 12 h y emite `shadowing.morning.alert`; Telegram lo escucha (Canvas no importa Telegram; mismo patrón que `autonomy.mode.changed`).

- **Cero llamadas extra al LLM.** Recalcular a las 06:00 serían 2 llamadas Gemini 3.1 Pro/día (hasta 8000 tokens de salida) contra el budget guard. Las fechas de entrega son absolutas: un resumen de 6 h sigue vigente.
- **Nunca un resumen vacío que parezca "todo tranquilo"**: tres casos explícitos — resumen real, "FALLÓ (motivo)", "NO se ejecutó" (sin corrida en 12 h, p. ej. pod caído a medianoche). Una corrida de hace más de 12 h no se presenta como la de anoche.
- **HITL `auto`**, con fila de audit (`system_cron` / `morning_alert_cron`). La alerta sale **antes** del audit: es informativa, no una acción irreversible, así que un audit bloqueado no debe silenciarla (a diferencia de la ejecución HITL, que es fail-closed, ADR 0010).
- **El resumen es contenido no confiable** (Canvas → LLM): se envía como texto plano **sin `parse_mode`** y truncado a 4096 caracteres.
- Sin recuperación de corridas perdidas: si el pod está caído a las 06:00 no se reenvía. Aceptable para una alerta informativa.

### Zona horaria: `TZ=America/Lima` en el Deployment
**Bug de despliegue hallado durante el diseño:** ningún pod de `jin-core` definía `TZ` (solo los CronJobs de backup traían `timeZone`), así que `@Cron('0 0 * * *')` y el reset diario del budget (`todayLocalDate`, getters locales) corrían en UTC: el "00:00 local" era 19:00 en Lima y las "06:00" habrían sido la 01:00. Se fija `TZ=America/Lima` (consistente con los backups; el owner está en Perú, BLUEPRINT §7.2 "Fase 1"). Mudarse a Canadá = cambiar una línea. *Decisión mía, revocable.*

### 9.6 — `ink-select-input`, no Inquirer.js
**Desviación deliberada de BLUEPRINT §8.3.** Inquirer toma el stdin por su cuenta y choca con Ink (ambos quieren el TTY). `ink-select-input` da los mismos menús de flechas dentro del árbol de Ink.

- **`jin inbox`**: bandeja navegable; aprobar/rechazar sin pegar un requestId. «Volver» y «No, cancelar» son lo resaltado por defecto: un Enter por reflejo nunca ejecuta una acción. La lógica de aprobar/rechazar se extrae a `hitl-actions.ts` y la comparten `jin approve|reject` y la bandeja (mismo tratamiento del 409 de Jin_Core#36).
- **`jin budget [unpause]`, `jin audit [n]`, `jin previews [stop [id]]`, `jin runs [id]`**: solo endpoints reales del contrato; tipos 100 % generados (regla de oro #11). Las acciones destructivas piden confirmación; `--yes` la salta en scripts. Sin TTY caen a texto.
- **Diferido, sin endpoint**: rotar tokens, forzar backup y ver logs. Core no puede tocar K8s (regla de oro #1); requeriría una tarea aparte vía Executor.

## Consecuencias

- Migración 0012 (retrocompatible) que aplica el Job de `Jin_Infra` con la imagen del Deployment.
- Una segunda dependencia de UI en la CLI (`ink-select-input@5`, compatible con Ink 4).
- Verificación de la CLI en una terminal real (pty): `ink-testing-library@3` no implementa `stdin.ref()/unref()/read()` que usa Ink 4.4, así que los tests lo parchean; la prueba con pty confirma que el parche no esconde un fallo real.

## Alternativas consideradas

- **Recalcular a las 06:00:** rechazado — doble costo de LLM al día para obtener lo mismo.
- **Enviar el resumen con Markdown:** rechazado — contenido controlable por un tercero (un anuncio de Canvas) llegaría interpretado al chat del owner.
- **Inquirer.js:** rechazado por el conflicto de stdin con Ink.
