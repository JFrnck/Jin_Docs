# Runbook: intervención manual del tier de escalado (Modal)

## Para quién es esto

Procedimiento operativo (`AGENTS.md` línea 394) para cuando el tier de escalado (Modal, `Jin_Executor/src/modal/modal.service.ts`, BLUEPRINT §4.5) se comporta mal — jobs que no terminan, cuota excedida, o el `App`/`Image` cacheado en `jin-executor` quedó en un estado roto.

## 0. Primero, diagnosticá desde qué lado viene el problema

- **`runCode` con `remote: true` cuelga o tarda demasiado:** revisá el dashboard de Modal (`modal.com`, cuenta asociada a `MODAL_TOKEN_ID`) — ahí se ve el sandbox real corriendo, sus logs, y su estado.
- **`runCode` falla con `ModalExecutionError` de forma consistente (no una vez):** probablemente el `App`/`Image` cacheado en el proceso de `jin-executor` quedó en un estado inconsistente — ver §1.
- **Cuota/rate limit de Modal excedida:** ver §2.

## 1. Limpiar el caché de `App`/`Image` de `jin-executor`

`ModalService` cachea el `App`/`Image` de Modal de forma perezosa **por el resto de la vida del pod** (ver el comentario en `modal.service.ts` — deliberado, para no resolver la imagen científica en cada llamada). Si la resolución falla, el propio código ya limpia el caché para permitir reintento en la siguiente llamada — pero si quedó en un estado roto que no se autocorrige:

```bash
kubectl -n jin-executor rollout restart deployment/executor
kubectl -n jin-executor rollout status deployment/executor
```

Esto fuerza a que el próximo `runCode` remoto resuelva el `App`/`Image` desde cero. Costo: la primera llamada tras el restart puede tardar varios minutos si la imagen `jin-data-science` no existía todavía (`ModalService` la construye on-demand la primera vez).

## 2. Cuota o rate limit de Modal excedida

Esto no lo resuelve un restart — es un límite de cuenta en Modal, no un bug del código.

1. Verificá el uso real en el dashboard de Modal (Usage/Billing de la cuenta).
2. Si es un pico legítimo (una tarea con mucho código científico pesado): esperar a que el límite se resetee, o subir el plan de Modal si es recurrente — fuera del alcance de este repo.
3. Si es runaway (algo está llamando `runCode` remoto en loop): el kill switch de `Jin_Core` (`BudgetGuardedModelRouter`/`KillSwitchService`) debería haber cortado el flujo de LLM que dispara las llamadas antes de que esto llegue a ser un problema de cuota — si no lo hizo, es un hallazgo real para `docs/security-audit-fase7.md`, no algo a resolver a mano acá.

## 3. Ajustar los hard caps (timeout/memoria) — requiere PR, no es "manual" en runtime

`remoteMaxTimeoutSeconds` (1800s) y `remoteMemoryLimitMiB` (4096) viven en `Jin_Executor/src/rbac/tool-whitelist.ts`, hardcodeados por tool — no hay ninguna variable de entorno ni flag de runtime para subirlos temporalmente. Si un caso de uso real necesita más, el cambio es un PR normal a `tool-whitelist.ts` (revisión humana, mismo criterio que cualquier cambio de `hitlLevel` — `AGENTS.md` 5.4), no una intervención de incidente.

## 4. Rotar credenciales de Modal

Si `MODAL_TOKEN_ID`/`MODAL_TOKEN_SECRET` se comprometieron: regenerar en el dashboard de Modal, y seguir el runbook de `rotate-secrets.md` (viven en Infisical desde Fase 8.1, identidad de máquina de `jin-executor`).
