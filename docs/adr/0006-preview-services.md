# ADR 0006 — Pods de servicio: preview apps expuestas bajo `*.jinserver.com`

## Contexto

BLUEPRINT §4 (ejecución aislada) y ADR 0003 definen el Executor como frontera única de contacto con Kubernetes, corriendo hoy solo pods run-to-completion. El owner pidió (2026-07-25) que los agentes puedan levantar procesos de larga vida con puerto expuesto — la regla de oro #10 ya anticipaba que esto sería contenido bajo `jinserver.com`, nunca `jeanfranck.com`.

## Decisiones

### 1. Origen del código: archivos inline, no git real (decisión del owner)

`PROMPTS.md` §5.5 sugiere "el código que el agente generó/clonó en su branch", pero ninguna tool existente le da a un agente la capacidad de escribir a un repo git real — mismo gap que ADR 0005 dejó documentado para `mergeAgentBranch`. `startPreviewService` recibe `files: Record<string, string>` inline; un init container los extrae de un tar.gz (empaquetado server-side) a un volumen compartido, sin ConfigMap (RBAC no lo permite) ni riesgo de inyección de shell: el alfabeto base64 excluye por construcción cualquier metacaracter, así que aunque el init container use `sh -c` para decodificar+extraer, el payload solo se expande como variable de entorno entre comillas — el comando en sí es un string fijo y literal. Clonar desde una branch real queda como extensión futura, condicionada a que exista una tool de edición de código.

### 2. Módulo nuevo (`src/preview-service/`), no extensión de `PodLifecycleService`

El modelo de ciclo de vida es opuesto al de `PodLifecycleService` (nunca destruye en un `finally`, no bloquea esperando un estado terminal) — forzarlo en la misma clase complicaría la máquina de estados existente sin necesidad real.

### 3. Kubernetes como store de estado, sin Postgres nuevo en Jin_Executor

El repo nunca tocó una base de datos. El TTL vive como annotation en el propio pod (`jin.io/expires-at`), el slug como otra (`jin.io/slug`); `listPreviewServices` lista por label (`jin.io/type=service`). Evita introducir una dependencia nueva (DB) para un estado que Kubernetes ya modela correctamente.

### 4. PVC de pnpm compartido, viable por ser clúster de un solo nodo

`ReadWriteOnce` bloquea montaje *entre nodos*, no *entre pods del mismo nodo*. La VM OCI de este proyecto es single-node — un PVC `local-path` (provisioner default de K3s, mismo criterio que el resto de PVCs del repo, sin `storageClassName` explícito) compartido resuelve el requisito de PROMPTS.md (evitar 30-90s de install en frío por preview) sin storage RWX nuevo.

### 5. `fs.inotify.max_user_watches`: sysctl de nodo, script de bootstrap

No es un sysctl namespaced apto para `securityContext.sysctls` sin privilegios de pod. Se sube una vez en la VM vía `scripts/bootstrap/07-sysctl-inotify.sh`, mismo patrón que `06-prepull-deno.sh`, documentado en el README como paso 9 del bootstrap.

### 6. NetworkPolicy de ingreso nueva, scoped por pod

`agents-sandbox` deniega `Ingress` por default (además de `Egress`, ver `k8s/base/network-policies/`). Sin una policy nueva permitiendo tráfico desde el namespace de Traefik (`kube-system`) al puerto expuesto, ningún pod de servicio sería alcanzable — el mecanismo de exposición fallaría en silencio tras el "éxito" aparente de crear el Service/IngressRoute. Se agrega `buildServiceIngressNetworkPolicy()` en Jin_Executor, aditiva sobre las policies de namespace existentes (mismo criterio que la egress policy de pods run-to-completion).

### 7. IngressRoute (CRD de Traefik) vía `CustomObjectsApi`, primer uso en el repo

Grupo `traefik.io`/`v1alpha1` (Traefik v3, bundleado por la versión de K3s pinneada). `@kubernetes/client-node@1.4.0` ya expone `createNamespacedCustomObject`/`deleteNamespacedCustomObject` — sin necesidad de actualizar la librería. Verificado con un CRD real registrado en el test de integración de K3s (`@testcontainers/k3s` arranca con `--disable=traefik`, así que se registra el `CustomResourceDefinition` de `IngressRoute` a mano en el test, sin controller de Traefik reconciliándolo — el apiserver igual valida y persiste el manifest real).

### 8. Certificate nuevo en `agents-sandbox`

El wildcard `*.jinserver.com` ya existe (namespace `jin`) pero su Secret no es visible en `agents-sandbox` (Secrets TLS son namespace-scoped, mismo criterio ya usado para duplicar el Certificate de `jeanfranck.com` entre `jin` y `observability`). Se declara un `Certificate` nuevo en `agents-sandbox` contra el mismo `ClusterIssuer: letsencrypt-dns01`.

### 9. Slug con entropía real

`<nombre-legible>-<6 chars aleatorios>` (`generateSlug()`), validado contra DNS-1035 (empieza con letra, ≤63 chars el label completo — un hint que arranca en dígito se prefija con `p-`). Sin login delante de una preview, la parte aleatoria es la única barrera contra quien adivine la URL — nunca un slug derivado solo del nombre del proyecto.

### 10. TTL en variables de entorno (`PREVIEW_SERVICE_*`), no un YAML nuevo

A diferencia de Jin_Core (que usa `config/*.yaml` para guardas operativas como `max_iterations_per_turn`), Jin_Executor no tiene ese patrón establecido — su convención propia es `env.schema.ts` (Zod) para límites operativos con default sensato (`DENO_IMAGE`, `AGENTS_SANDBOX_NAMESPACE`). `PREVIEW_SERVICE_DEFAULT_TTL_SECONDS`/`PREVIEW_SERVICE_MAX_TTL_SECONDS`/`PREVIEW_SERVICE_MAX_CONCURRENT` siguen esa convención existente en vez de importar el patrón yaml de otro repo. Cap duro de 24h validado también en código (`Math.min`), no solo en el schema.

### 11. RBAC ampliado (`services`, `ingressroutes.traefik.io`), `ResourceQuota`/`LimitRange` intactos

Restricción explícita de PROMPTS.md §5.5: no ampliar los límites de recursos del namespace en este PR. Los pods de servicio compiten por el mismo presupuesto de recursos que los pods run-to-completion (request 256Mi/250m, limit 1Gi/1000m — dentro del máximo de 2Gi/1500m). Cambio de RBAC hecho por Claude Code en Jin_Infra pese a ser normalmente área de Antigravity — frontera de seguridad, mismo precedente que backups en Fase 1.2 (ver `PROMPTS.md` §5.5 y nota de coordinación en `STATUS.md`).

### 12. `ExecutorToolDefinition` como discriminated union

`src/rbac/tool-whitelist.ts` (Jin_Executor) separa `RunToCompletionToolDefinition` (con `maxTimeoutSeconds`/`remoteMaxTimeoutSeconds`/`remoteMemoryLimitMiB`) de `ServiceToolDefinition` (`isServiceTool: true`, sin esos campos) en vez de campos opcionales sueltos — permite narrowing de TypeScript real (`isRunToCompletionTool()`) sin non-null assertions, y hace estructuralmente imposible que `PodLifecycleService`/`ModalService` procesen una tool de servicio por error (fail-safe explícito, verificado con test).

### 13. `@nestjs/schedule` como dependencia nueva en Jin_Executor

Necesaria para el reaper (`@Cron`). Ya vetted en Jin_Core desde Fase 4.1 — riesgo bajo, señalado explícitamente por ser la primera dependencia de este tipo en Jin_Executor.

## Consecuencias

- Primera vez que Jin_Executor expone un CRD de terceros (Traefik) — precedente para futuras integraciones de red.
- Jin_Executor sigue sin tocar Postgres — el estado de servicios vive 100% en Kubernetes, consistente con el resto del repo.
- Deuda documentada: origen del código por git real queda pendiente de una tool de edición de código que no existe hoy (mismo patrón que `mergeAgentBranch`, ADR 0005 punto 9).
- La VM sigue siendo un único punto de fallo de storage (PVC `local-path` no es HA) — aceptado, coherente con el resto de la infra actual (Postgres/Redis tampoco son HA en Fase 1).
- El agent loop (Fase 5.1) puede invocar `startPreviewService`/`stopPreviewService`/`listPreviewServices` sin cambios adicionales: son tools declaradas en `registry.ts` como cualquier otra, ejecutadas vía `ToolExecutorRegistry` (Fase 4.2).

## Alternativas consideradas

- **Clonar desde git real (texto literal de PROMPTS.md):** descartado para v1 — requiere resolver primero cómo un agente escribe código a un repo real (credenciales, host), alcance bastante mayor que esta fase.
- **Extender `PodLifecycleService` en vez de un módulo nuevo:** descartado — el modelo de ciclo de vida es opuesto (persistente vs. run-to-completion + destroy garantizado), forzarlo complica la clase existente sin necesidad.
- **PVC RWX (Longhorn/NFS) para el pnpm store:** descartado — infraestructura nueva no justificada cuando el clúster de un solo nodo ya resuelve el caso con RWO.
- **Guardar el estado de servicios en Postgres (vía Jin_Core, no Jin_Executor):** descartado — rompería la frontera de responsabilidad (Jin_Executor es el único que habla con K8s; el estado real YA vive ahí) y agregaría una fuente de verdad redundante que podría desincronizarse del clúster real.
- **`config/preview-services.yaml` (imitando el patrón de Jin_Core):** descartado — Jin_Executor no tiene ese patrón establecido; introducirlo solo para esta fase habría sido inconsistente con el resto del repo (`env.schema.ts` ya cubre exactamente este tipo de guarda operativa).
