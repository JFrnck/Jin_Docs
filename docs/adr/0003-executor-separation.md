# ADR 0003 — Executor: frontera de ejecución aislada

> Nota de numeración: PROMPTS.md 2.3 sugería `0002-executor-separation.md`, pero 0001 (niveles HITL) y 0002 (`request_id`/`pending_approvals`) ya estaban tomados por decisiones de la Fase 2.2. Esta es la 0003.

## Contexto

BLUEPRINT §4.2-4.4 y AGENTS.md §5.3 establecen que Jin_Executor es el único proceso autorizado a hablar con Kubernetes, y que corre código LLM-generado en pods Deno efímeros bajo demanda. La Fase 2.3 implementa ese microservicio: validación RBAC contra una whitelist de tools, construcción y ciclo de vida de los pods, y el endpoint `POST /execute`.

Durante la implementación surgieron varias decisiones no cubiertas explícitamente por BLUEPRINT/PROMPTS, más un hallazgo de seguridad real verificado empíricamente contra un clúster K3s de prueba.

## Decisiones

### 1. Whitelist propia del Executor, separada del registry de Core

`src/tools/registry.ts` de Jin_Core declara `hitlLevel` (nivel de aprobación humana). El Executor necesita un concepto distinto — qué identificadores de tool pueden ejecutar código y con qué `egressWhitelist` — así que declara su propio registry estático (`src/rbac/tool-whitelist.ts`), sin paquete compartido entre repos (AGENTS.md 4.5). Arranca con una sola tool stub, `runCode`, sin egreso.

### 2. Limitación conocida: whitelist de egreso por dominio no resuelve a CIDR todavía

AGENTS.md 5.5 exige que el Executor aplique NetworkPolicies según el `egressWhitelist` (dominios) de cada tool. Un `NetworkPolicy` de Kubernetes estándar no entiende nombres de dominio, solo IPs/CIDRs/selectores — K3s usa Flannel por defecto, sin soporte FQDN (a diferencia de Cilium). Como ninguna tool actual necesita egreso real, se implementó el mecanismo (`buildPodNetworkPolicy` acepta CIDRs ya resueltos) pero **no** la resolución dominio→CIDR: si una tool futura declarara un `egressWhitelist` no vacío, `PodLifecycleService` lanza `UnresolvedEgressWhitelistError` (501) en vez de conceder egreso de forma incorrecta o adivinada. Resolver esto (IPs resueltas al crear el pod, o un proxy de egreso con ACL por dominio) queda pendiente para cuando exista la primera tool real con esta necesidad (Fase 5).

### 3. RBAC del ServiceAccount: extensión mínima necesaria más allá del texto literal de BLUEPRINT

BLUEPRINT 4.2 describe el Role como "create/get/list/delete de Pods, nada más". Para cumplir con el punto 2 (crear una NetworkPolicy por pod cuando haga falta) el ServiceAccount necesita además `create`/`delete` sobre `networkpolicies` en el namespace `agents-sandbox` — sin este permiso, AGENTS.md 5.5 sería imposible de cumplir. Es una extensión necesaria, no una desviación: sigue sin acceso a Secrets, ConfigMaps, Deployments, ni otros namespaces. **Pendiente:** actualizar el manifest de Role en `Jin_Infra/k8s/base/executor/` (PR aparte, coordinado por STATUS.md, según indica PROMPTS.md 2.3).

### 4. `deno eval` es inadecuado para el sandbox — usar `deno run`

Verificado empíricamente contra un K3s real (Deno 2.9.3): **`deno eval` tiene acceso implícito a todos los permisos** — ignora por completo `--allow-net` y cualquier otro flag de permisos, sin importar si se pasan o no (confirmado con la propia ayuda de la CLI: *"This command has implicit access to all permissions"*). Usarlo habría dejado el sandbox completamente abierto pese a la apariencia de estar restringido — un hallazgo de seguridad real, no cosmético.

La corrección: usar `deno run`, que sí respeta el modelo deny-by-default de Deno (verificado: sin `--allow-net` un intento de red falla con `NotCapable`; con el flag, se permite). `deno run` necesita un módulo (archivo o URL), no código inline. Para evitar dos alternativas peores — un ConfigMap con el código (prohibido: el RBAC del Executor no tiene acceso a ConfigMaps, BLUEPRINT 4.2) o una shell para escribirlo a un archivo temporal (la imagen distroless no tiene shell, y reintroduciría superficie de inyección) — el código se pasa como un `data:` URL en base64: `deno run --allow-net=... data:application/typescript;base64,<...>`. `args` de Kubernetes nunca pasa por una shell, así que esto preserva la propiedad de "sin riesgo de inyección" mientras corrige el sandboxing real.

### 5. Testing: K3s real vía testcontainers, no mocks del cliente de Kubernetes

Se decidió (con el owner) verificar el aislamiento de red con un clúster K3s real (`@testcontainers/k3s`) en vez de mockear `@kubernetes/client-node`. Un mock solo verificaría "se llamó a la API con estos parámetros", no que el aislamiento realmente funcione — y es precisamente la propiedad de seguridad más crítica de todo el diseño (BLUEPRINT: "es la frontera de seguridad central del proyecto"). El costo es CI más lento (~75s con la imagen pre-pulleada) — aceptado explícitamente.

Hallazgo operativo durante esta verificación: la imagen de Deno puede tardar >100s en descargarse dentro de un containerd anidado (K3s-en-Docker) sin pre-pull — el mismo problema, a menor escala, que motiva `scripts/bootstrap/06-prepull-deno.sh` en producción (BLUEPRINT 4.4). El test-support de Executor replica ese pre-pull (`ctr --address /run/k3s/containerd/containerd.sock -n k8s.io images pull <imagen>`) antes de correr las aserciones.

## Consecuencias

- El sandbox de ejecución es real: verificado con K3s real, no solo con el diseño en el papel.
- Queda un PR pendiente en Jin_Infra para la extensión de RBAC (punto 3) antes de que el mecanismo de NetworkPolicy por pod pueda usarse en producción — hoy es código muerto en la práctica porque ninguna tool lo activa.
- La resolución dominio→CIDR (punto 2) es deuda técnica explícita y deliberada, no un olvido: falla ruidosamente en vez de fallar en silencio.

## Alternativas consideradas

- **Mockear `@kubernetes/client-node` para todos los tests:** más rápido, pero no habría detectado el hallazgo del punto 4 (que solo se manifiesta contra un runtime Deno real) ni verificado el aislamiento de red real.
- **ConfigMap para pasar el código a `deno run`:** descartado — viola la restricción explícita de RBAC de BLUEPRINT 4.2.
- **Imagen Deno no-distroless + shell + archivo temporal:** descartado — reintroduce superficie de inyección de shell y aumenta el tamaño/superficie de ataque de la imagen sin necesidad, cuando el `data:` URL resuelve el mismo problema sin ninguna de las dos desventajas.
