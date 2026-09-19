# Auditoría de seguridad — Fase 7.3

**Fecha:** 2026-09-13. **Alcance:** las 12 reglas de oro de `BLUEPRINT.md` §15, verificadas contra el código real de los 6 repos (no de memoria ni de documentación), con RBAC/NetworkPolicies de `Jin_Infra` revisados en detalle. Criterio del BLUEPRINT (Fase 7): el sistema debe aguantar 7 días autónomo sin intervención salvo aprobaciones HITL.

**Cierre honesto:** ese criterio final **no se puede marcar como cumplido hoy** — se valida en operación real, y el deploy real está pospuesto hasta terminar el roadmap completo (decisión del owner, 2026-08-07). Lo que sigue es una verificación de que el sistema está **instrumentado y listo para esa validación**, no una certificación de que ya ocurrió.

## Regla por regla

### #1 — jin-core jamás toca el socket de Docker/K3s

Verificado: `grep -rl "dockerode\|@kubernetes/client-node" Jin_Core/src Jin_Core/package.json` → sin resultados. Todo lo que necesita hablar con K8s vive exclusivamso en `Jin_Executor` (AGENTS.md 5.3). **Cumple.**

### #2 — Ningún secreto en código, env, ni Git — todo en Infisical

Implementado en Fase 8.1: [Jin_Core #29](https://github.com/JFrnck/Jin_Core/pull/29), [Jin_Executor #11](https://github.com/JFrnck/Jin_Executor/pull/11), [Jin_Infra #12](https://github.com/JFrnck/Jin_Infra/pull/12) — `loadSecrets()` carga las 15 claves reales desde Infisical al startup. Excepción documentada y aceptada: `DATABASE_URL`/`REDIS_URL` y las credenciales de la identidad de máquina de Infisical siguen en un Secret de K8s (huevo-gallina de infra, ver `BLUEPRINT.md` §11). **Cumple, con la excepción ya documentada.**

### #3 — Ningún backup no probado cuenta como backup

Verificado: `Jin_Infra/scripts/backup/verify-restore.sh` (leído completo). Cubre los 3 backups reales: Postgres (`pg_restore` + conteo de filas por tabla), Redis (`redis-check-rdb` + un `redis-server` efímero real respondiendo `PING`/`DBSIZE`), y `memory.db` (`PRAGMA integrity_check` + conteo de tablas y de filas en `vec0`). **Cumple** — y de hecho ya cumplía desde [Jin_Infra PR #8](https://github.com/JFrnck/Jin_Infra/pull/8) (mergeado 2026-08-05).

**Hallazgo de proceso, corregido en este PR:** `STATUS.md` seguía marcando los puntos 15 y 25 de `RECOMENDACIONES.md` como abiertos (`[ ]`) pese a que el PR #8 que los resuelve está mergeado desde hace más de un mes. Corregido a `[x]` con la cita del PR real.

### #4 — El hitlLevel de una tool jamás lo decide el LLM

Verificado en código (`Jin_Core/src/hitl/classifier.ts::classifyToolCall`): el parámetro `_inputs` está prefijado con `_` a propósito y nunca se lee — el nivel sale únicamente de `getToolDefinition(toolName).hitlLevel` (el registry estático). Verificado además como contrato de test explícito en el golden set de Fase 8.2 ([Jin_Core #30](https://github.com/JFrnck/Jin_Core/pull/30), `corpus-classifier.spec.ts`, 19 tests): 3 tools `confirm` reales clasificadas contra las 6 entradas de `hitl-escalation` (payloads que intentan afirmar una reclasificación) — el nivel nunca cambia. **Cumple.**

### #5 — Ningún modelo LLM hardcoded — todo por `models.yaml`

Verificado: `grep -rn "'claude-\|'gpt-\|'gemini-" Jin_Core/src` (excluyendo `model-provider/`/`config/models.yaml`) → sin resultados. `models-config.schema.ts` carga y valida `config/models.yaml` con Zod, fail-fast si es inválido. **Cumple.**

### #6 — Ningún input externo entra al prompt sin envolver

Verificado: `wrapUntrustedContent`/`sanitizeForIndexing` (`Jin_Core/src/security/injection-sanitizer.ts`, ADR 0004) son el único punto de entrada de contenido externo al contexto de un LLM. El golden set de Fase 8.2 (115 tests, 50 payloads en 8 categorías) **es** la verificación de "intentos activos de prompt injection contra el agent loop" que esta misma regla pide — no se reconstruye acá, se cita como evidencia ya cerrada: `corpus-sanitizer.spec.ts` (84 tests, escapado + nonce), `agent-pipeline.spec.ts` (12 tests, el loop completo con un correo hostil real que intenta hacer creer que una aprobación `confirm` ya ocurrió). **Cumple.**

### #7 — Toda tool destructiva es `dual-confirm`. Cuando dudes, `dual-confirm`

Verificado el registry completo (`Jin_Core/src/tools/registry.ts`, 17 tools): **ninguna usa `hitlLevel: 'dual-confirm'` hoy** — el nivel máximo en uso es `confirm` (5 tools: `sendEmail`, `deleteCalendarEventFuture`, `runCode`, `resolveAgentConflict`, `mergeAgentBranch`, `startPreviewService`).

**Hallazgo, no corregido en este PR (requiere decisión del owner):**

- `sendEmail` es la tool más candidata a `dual-confirm` bajo el criterio de esta regla: es **irreversible** (no existe "des-enviar" un correo) y actúa con la identidad del owner ante un tercero — el tipo de acción para el que la regla dice explícitamente "cuando dudes, elegí dual-confirm". Recomendación: subir `sendEmail` a `dual-confirm`.
- `mergeAgentBranch` hoy es un 501 documentado (sin tool que produzca una branch real todavía, Fase 9.2 la habilita) — no representa riesgo activo, pero cuando Fase 9.2 la implemente, mergear código a `main` de forma autónoma es exactamente el tipo de acción que esta regla anticipa. Recomendación: reevaluar `dual-confirm` **al implementar Fase 9.2**, no antes.
- `runCode` y `startPreviewService` se evaluaron y se consideran proporcionados en `confirm`: ambos corren en sandbox aislado (Deno sin red / pod con TTL duro y sin secretos), la contención técnica ya reduce el impacto de un error — subir a `dual-confirm` sería fricción sin una ganancia de seguridad clara.

Ninguno de estos cambios se aplica en este PR: `AGENTS.md` 5.4 exige que un cambio de `hitlLevel` pase por PR con revisión humana **y** una aprobación `dual-confirm` en el sistema en producción para tomar efecto — cambiarlo unilateralmente en una auditoría violaría la regla que la propia auditoría está verificando. Queda como recomendación explícita para que el owner decida.

### #8 — Toda decisión HITL incluye los inputs externos que influyeron

Verificado: `summarizeUntrustedSources()` (`injection-sanitizer.ts`) se llama en `AgentService.handleRealToolCall` antes de cada `createPendingApproval`, y `agent.service.spec.ts` tiene un test explícito ("tool confirm precedida por una tool auto en el mismo turno: createPendingApproval recibe externalInputsSummary con la traza real") que lo verifica end-to-end. **Cumple.**

### #9 — El timeout nunca aprueba automáticamente

Verificado: `Jin_Core/src/hitl/timeout.service.ts::processPending` — el `switch` sobre `timeoutBehavior` solo tiene los casos `'discard'` y `'escalate-warning'`. No existe ningún case `'approve'` ni default que apruebe. **Cumple.**

### #10 — Contenido de agentes solo en el dominio sandbox

Verificado: `Jin_Executor/src/preview-service/preview-service.service.ts` construye la URL de cada preview con `` `https://${slug}.jinserver.com` `` — hardcoded, ningún parámetro permite sustituir el dominio. Los `Certificate` de cert-manager (`Jin_Infra/k8s/base/cert-manager/certificates.yaml`) están correctamente separados: `wildcard-jeanfranck-com` para la zona confiable, `wildcard-jinserver-com` para la zona sandbox, en Secrets de TLS distintos. La única `IngressRoute` estática sobre `jeanfranck.com` (`jin-core-api`) apunta a `jin-core` con `PathPrefix(/api)`, nunca a un pod de `agents-sandbox`. **Cumple.**

### #11 — Ningún tipo se copia a mano entre repos

Verificado en los 4 CI de repos de app: `Jin_Core`/`Jin_Executor` corren `pnpm run generate:contract` (generan `contracts/openapi.json` desde el código); `Jin_Web`/`Jin_CLI` corren `pnpm run generate:api` (generan tipos desde ese OpenAPI). **Cumple.**

### #12 — Blueprint y código nunca difieren en silencio

Dos hallazgos reales, ambos corregidos en este PR:

1. **`AGENTS.md` §5.5 no aclaraba dónde vive `egressWhitelist`.** El texto decía "cada tool declara los dominios... en `egressWhitelist`" sin decir que ese campo vive en el registry propio de `Jin_Executor` (`src/rbac/tool-whitelist.ts`), no en el de `Jin_Core` (`src/tools/registry.ts`, que solo declara `hitlLevel`) — son registries independientes por diseño (AGENTS.md 4.5, sin paquetes compartidos entre repos). El código ya era correcto y fail-safe (`UnresolvedEgressWhitelistError` si una tool declarara egreso real antes de que exista resolución dominio→CIDR, ADR 0003 punto 2) — era una imprecisión de redacción, no un bug. **Corregido:** una línea aclaratoria en `AGENTS.md` §5.5.

2. **RBAC del Executor: falta `pods/log` (ver sección siguiente).** Bug real, no solo de redacción — corregido en el código de infra, no solo documentado.

## Revisión de RBAC y NetworkPolicies

**Metodología:** cada verbo/recurso declarado en `Jin_Infra/k8s/base/executor/role.yaml` se comparó contra cada llamada real a la API de Kubernetes en `Jin_Executor/src/k8s/k8s.service.ts`.

**Hallazgo real, corregido en este PR:** el `Role` otorgaba `get`/`list`/`create`/`delete` sobre `pods`, pero `K8sService.getPodLogs()` (llamado por `PodLifecycleService` al final de cada `runCode`, para devolver el resultado) usa `readNamespacedPodLog`, que en RBAC de Kubernetes requiere el recurso **`pods/log`** — un subrecurso distinto de `pods`, no cubierto por el rule existente. En un clúster real, cada `runCode` habría creado y corrido el pod correctamente, pero **habría fallado con 403 al intentar leer el resultado** — el tipo de bug que solo se manifiesta contra RBAC real, nunca en local. Corregido: nuevo rule `resources: ["pods/log"], verbs: ["get"]` en `role.yaml`.

**Por qué ningún test lo atrapó:** los tests de integración de `Jin_Executor` (`*.integration.spec.ts`, K3s real vía `@testcontainers/k3s`) usan el kubeconfig admin del contenedor, no un token vinculado al `ServiceAccount`/`Role` reales de `executor-agents-sandbox`. **Recomendación, no implementada en este PR** (ampliaría el alcance más allá de esta auditoría): que al menos un test de integración cree el `Role`+`RoleBinding`+`ServiceAccount` reales y genere un token acotado a ellos, para que el propio CI atrape esta clase de gap en el futuro.

**Resto del `Role`:** `services` (`create`/`get`/`list`/`delete`) y `ingressroutes.traefik.io` (`create`/`get`/`list`/`delete`) — el código de Executor hoy solo ejercita `create`/`delete` de ambos (`PreviewServiceLifecycleService`); `get`/`list` están otorgados pero sin consumidor todavía (el comentario del propio `role.yaml` ya lo anticipa como uso futuro de `PreviewServiceLifecycleService.list()`, que hoy en realidad lista pods por label, no services — dato menor, no un riesgo: no es sobre-privilegio explotable, es margen documentado). `networkpolicies` (`create`/`delete`, sin `get`/`list`) coincide exactamente con lo que usa `K8sService` (`createNamespacedNetworkPolicy`/`deleteNamespacedNetworkPolicy`). Ningún acceso a `Secrets`, `ConfigMaps`, ni `Deployments` en ningún rule — confirmado, coincide con BLUEPRINT §4.2. **NetworkPolicies de namespace** (`Jin_Infra/k8s/base/network-policies/agents-sandbox.yaml`): `default-deny-all` (Ingress+Egress) + `allow-dns-egress` — confirmado que las policies por-pod que crea el Executor (`network-policy.builder.ts`) son aditivas sobre estas, nunca las reemplazan.

## Nota aparte, fuera del alcance de esta auditoría

Al encontrar que `STATUS.md` tenía 2 checkboxes desactualizados en el mismo bloque de "Fase 0.1 debug #1" (puntos 15 y 25, corregidos arriba), vale la pena que alguien verifique si los puntos 14/16(mitad CLI)/20(mitad CLI)/20.b del mismo bloque (`Jin_CLI`, ownership de Antigravity) tienen el mismo problema — no se verificó acá porque cruza la frontera de "no se toca desde esta sesión sin negociación previa" (`Jin_CLI/CLAUDE.md`).

## Resumen

| # | Regla | Estado |
| --- | --- | --- |
| 1 | jin-core no toca Docker/K3s | ✅ Cumple |
| 2 | Secretos solo en Infisical | ✅ Cumple (excepción documentada) |
| 3 | Backup probado | ✅ Cumple (tracking corregido) |
| 4 | hitlLevel nunca lo decide el LLM | ✅ Cumple |
| 5 | Modelo nunca hardcoded | ✅ Cumple |
| 6 | Input externo siempre envuelto | ✅ Cumple |
| 7 | Destructivo = dual-confirm | ⚠️ Recomendación abierta (sendEmail), sin cambiar sin el owner |
| 8 | external_inputs_summary en HITL | ✅ Cumple |
| 9 | Timeout nunca aprueba | ✅ Cumple |
| 10 | Contenido de agentes solo en sandbox | ✅ Cumple |
| 11 | Contratos generados, no copiados | ✅ Cumple |
| 12 | Blueprint/código nunca difieren en silencio | ✅ Corregido (2 hallazgos) |

**Sin hallazgos críticos abiertos.** Un hallazgo real corregido (RBAC `pods/log`), un hallazgo de redacción corregido (`AGENTS.md` §5.5), un hallazgo de tracking corregido (`STATUS.md` puntos 15/25), y una recomendación de diseño (`sendEmail` → `dual-confirm`) que queda para decisión explícita del owner.
