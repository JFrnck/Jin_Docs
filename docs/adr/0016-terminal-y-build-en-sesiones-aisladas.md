# ADR 0016 — Terminal y builds (Vite/npm) en sesiones aisladas

## Contexto

El editor del iPhone (ADR 0015) publica sitios **estáticos**. El owner pidió poder crear proyectos como Vite y tener una terminal:

- Un pod hoy no tiene internet (default-deny + DNS). `npm install` y `vite build` no pueden correr.
- `agents-sandbox` no puede abrir salida a `registry.npmjs.org` con una `NetworkPolicy`: solo admite CIDRs y el registro está detrás de una CDN con IPs cambiantes (la resolución dominio→CIDR no existe, ver ADR 0003).
- Ejecutar comandos arbitrarios es lo más delicado del sistema. Toca `Jin_Executor` (revisión línea por línea, AGENTS.md §2.1).

## Decisiones

### Una sesión de terminal es un pod propio, no el de un preview

- **Sesión:** un pod de larga vida en `agents-sandbox` (imagen `node:22-alpine`, sin root, PSA `restricted`, sin token de ServiceAccount) que solo espera comandos. Tiene TTL (1 h por defecto, tope 4 h), lo destruye el reaper, y hay **una sola** a la vez.
- **Espacio de trabajo:** `emptyDir` (no el PVC compartido `pnpm-store`, para que un paquete malicioso no envenene la caché de otros pods).
- **Comandos:** el Executor los ejecuta con la API `exec` de Kubernetes, uno por vez, con `timeout` (120 s por defecto, tope 600 s) y tope de salida (512 KB). Cada comando corre en una shell nueva; el directorio actual se conserva entre comandos con un archivo en `/tmp`, para que `cd` funcione.
- **(Superado el 2026-09-29: ver "Ampliación 2026-09-29" abajo, que agrega una terminal interactiva con PTY.)** Este modo, un comando por vez con salida en vivo, se conserva como "Comandos" y lo siguen usando los servidores en segundo plano, la vista previa y los archivos. Cubre `npm install`, `npm run build`, `ls`, `cat`, `node`; no cubre programas que piden teclas (`npm create vite` sin flags, `vim`, un asistente).

### La única salida de red es un proxy de npm dentro del clúster

- Un **Verdaccio** propio (namespace `registry-proxy`) que solo habla con el registro de npm. Los pods de terminal solo pueden salir a ese servicio (una `NetworkPolicy` por sesión, creada por el Executor) y a DNS. Ningún otro destino de internet.
- El proxy no acepta publicar ni registrar usuarios.
- Un `postinstall` de un paquete corre dentro del pod aislado y solo alcanza el proxy.
- **Riesgo residual, aceptado:** el proxy es un canal de exfiltración de ancho de banda mínimo (pedir un paquete cuyo nombre codifica datos). Lo que hay en el pod es el proyecto del propio owner, sin secretos: los pods no reciben variables de entorno ni tokens. (Excepción opt-in por sesión desde 2026-09-29: Claude Code, con su propio proxy y un token del owner dentro del pod; ver ADR 0017.)
- Alternativas descartadas: abrir egreso por CIDR de una CDN (frágil e inseguro) y un DNS/egress FQDN (necesita otro CNI).

### El modelo no tiene terminal

- Las tools son **virtuales** (mismo mecanismo que `autonomyModeChange`): no están en `TOOL_REGISTRY`, así que el LLM no puede verlas ni invocarlas (`UnknownToolError`). Los comandos los escribe el owner.
- Sin comandos generados por un agente no hay vía de inyección de prompt hacia la terminal.

### HITL

- **`startTerminalSession`: `confirm` siempre**, con aprobación pendiente `owner:api`. Los modos de autonomía **no** la relajan: crea un pod con salida a un registro.
- **`exposeTerminalSession`: `confirm` siempre.** Publica un link público del build. Es el mismo riesgo que `startPreviewService`.
- **Los comandos dentro de una sesión aprobada no piden aprobación uno por uno**: los teclea el owner, dentro de un sandbox sin más red que el proxy. Pedir aprobación por cada `ls` haría la terminal inservible, y aprobar sin leer es peor que no preguntar. La aprobación es de la sesión.
- **Cada comando queda en el audit** (`runTerminalCommand`, actor `owner:terminal`) **antes** de ejecutarse: hash y primeros 120 caracteres. Si el audit falla, el comando no corre (fail-closed).

### Publicar un build

`exposeTerminalSession` levanta el servidor estático fijo de Jin dentro de la propia sesión, apuntando al directorio del build (`dist` por defecto), y crea el `Service` + `IngressRoute` como un preview. Así no hay que copiar el build a otro pod (un env var de K8s no pasa de ~128 KB, y un build de Vite puede ser mayor). El link vive lo que el TTL de la sesión.

### Traer y llevar archivos

- Exportar el espacio de trabajo al editor (para que lo que generó `npm create vite` aparezca en el proyecto del iPhone): archivos de texto, sin `node_modules`, `.git` ni `dist`, con los mismos topes que Publicar (50 archivos, 256 KB).
- Enviar el proyecto del editor a la sesión (al abrirla, y a pedido).

### Vista previa en vivo de un servidor de la sesión

Para ver una app que corre en un puerto (`npm run dev` en el 5173, un backend en el 3000) dentro de la app:

- **Servidor en segundo plano:** se lanza en su propio grupo de procesos, con la salida a un archivo, y sobrevive al comando (el ejecutor de comandos sí mata todo al terminar). Se puede listar, ver su log y detener (se mata el grupo entero). Lanzarlo y detenerlo son un comando más del owner: quedan en el audit antes (fail-closed) y no piden aprobación, porque no exponen nada afuera.
- **Camino privado, no un link público:** app → Core (`/api/terminal/sessions/:id/preview/:puerto/*`, con el JWT del owner) → Executor → API server (`pods/proxy`) → pod. La vista web de la app no conoce el token: un `WKURLSchemeHandler` nativo lo agrega a cada petición. No se reenvían cookies ni el JWT al servidor, ni `set-cookie` de vuelta; nunca se cachea.
- **El Executor no alcanza los pods directamente** (su NetworkPolicy excluye los CIDRs del clúster): todo pasa por `pods/proxy`, con verbos acotados a `agents-sandbox`. La ruta se valida antes de tocar la red (sin `..`, ni escapado, ni barra invertida, ni caracteres de control) para que no pueda salir del prefijo del pod y llegar a otra ruta del API. El `Host` es `localhost:<puerto>` (Vite rechaza otros).
- **Encontrado probando Vite de verdad:** el API server **reescribe los enlaces del HTML** (`src="/@vite/client"` sale como `/api/v1/namespaces/…/proxy/@vite/client`) y la página dejaba de funcionar. El Executor deshace esa reescritura (cuerpo HTML y `Location`).
- **No se audita cada petición** (una página son decenas): lo que habilita la vista previa, el servidor, ya quedó en el audit.
- **Limitaciones:** la recarga automática por WebSocket (HMR) no pasa (se recarga a mano con ↻); las redirecciones se siguen con una página que navega; las URL absolutas a `localhost` dentro de la app del owner no se traducen.

### Todos los pods, en una sola lista

`Más → Pods` junta las apps publicadas (por un agente o desde el editor) y la sesión de terminal.

- **Auditar:** cada pod guarda la aprobación que lo originó (`jin.io/request-id`). Ese id lo pone **quien ejecuta** (`ToolExecutionContext`), nunca el payload, que puede escribir el modelo: un `requestId` dentro del payload se ignora o se rechaza. `GET /api/audit?requestId=` devuelve el rastro de esa acción. Los pods anteriores a este campo no tienen enlace.
- **Editar:** "Al editor" copia los archivos de texto de un pod vivo a un proyecto nuevo (`GET /api/preview-services/:id/files`). Es una copia, con los mismos topes de Publicar, y **queda en el audit antes de leer** (fail-closed). El pod es código que escribió un agente: lo leído es un dato no confiable, y el programa que lo lee no sigue enlaces simbólicos ni sale del espacio de trabajo. Volver a publicar crea otro pod con otro link.

## Consecuencias

- **Jin_Infra:** namespace `registry-proxy` con Verdaccio y sus `NetworkPolicy`, permiso `pods/exec` para el Executor, y la cuota de `agents-sandbox` (hoy `limits.cpu: 1000m`, que alcanza para **un** pod) sube para que una sesión y un preview convivan.
- **Jin_Executor:** módulo `terminal`, con su contrato OpenAPI.
- **Jin_Core:** módulo `terminal`, dos tools virtuales, endpoints `/api/terminal/*` y audit de cada comando.
- **Jin_iOS:** pantalla de terminal, importar y exportar archivos, y "Publicar build".
- **No verificado hasta desplegar:** el `exec` y el proxy contra el clúster real (el Role real, la cuota, la resolución de DNS al proxy). Los tests del Executor usan un cliente K8s simulado; los de K3s con testcontainers necesitan Docker.

## Fuera de esta tanda

- ~~Terminal interactiva con PTY.~~ (Hecho el 2026-09-29, ver la ampliación.)
- Que el modelo use la terminal.
- Vite en modo desarrollo (`vite dev` con recarga): servir un servidor de desarrollo detrás del ingress necesita `allowedHosts` y HMR por WebSocket.

## Ampliación (2026-09-28): un workspace por proyecto, con disco propio

### Contexto

"Varias sesiones a la vez" quedó fuera de la tanda original, pero el owner
pidió después poder tener hasta ~10 proyectos guardados (dependencias
instaladas, archivos tal como quedaron) sin que eso signifique 10 pods
corriendo: la VM (2 vCPU/12GB, Oracle Always Free) no da para eso, y
tampoco es lo que quería — solo 1-3 pods activos a la vez, el resto
"apagados pero guardados".

La primera idea del owner fue un cron que duerma cada pod. No sirve:
Kubernetes reserva CPU/memoria por el `request` del pod, no por su uso
real — un pod dormido sigue ocupando su cupo en la `ResourceQuota`. La
solución correcta es separar **disco** de **cómputo**.

### Decisión: `PersistentVolumeClaim` por proyecto, pod bajo demanda

- **Disco:** un PVC (`local-path`, `ReadWriteOnce`, 3Gi por defecto) por
  proyecto, nombrado `terminal-ws-<workspaceId>`. `workspaceId` es el
  mismo id que el proyecto tiene en el editor del iPhone (`CodeProject.id`,
  un UUID) — nombra el disco Y el pod, así que se valida con el mismo
  regex de UUID en las tres puntas (Executor, Core, y de origen en la
  app) antes de tocar cualquier nombre de recurso de Kubernetes.
- **Pod:** se crea al abrir el proyecto (o al reanudarlo) y se destruye al
  cerrar o por inactividad (`TERMINAL_IDLE_TIMEOUT_SECONDS`, 30 min por
  defecto) — el disco **nunca** se toca al destruir el pod.
- **Tope de proyectos con disco:** `TERMINAL_MAX_WORKSPACES` (10 por
  defecto); el tope de pods corriendo a la vez sigue siendo
  `TERMINAL_MAX_CONCURRENT` (independiente — es la cuota de la VM la que
  manda ahí, ajustable sin tocar código).
- **`status: 'stopped'`** (disco sin pod) es el reposo normal de un
  proyecto, no un error. `list()` ahora devuelve todos los workspaces del
  owner, corriendo o no.

### HITL: sin cambios de nivel, ampliado a "por workspace"

Crear un pod con salida al proxy de npm es el mismo riesgo exista o no ya
el disco del proyecto — no hay razón de seguridad para tratar "reanudar"
distinto de "abrir por primera vez". **`startTerminalSession` (ahora
`start`/reanudar de un workspace) sigue en `confirm` fijo**, sin que
ningún modo de autonomía lo relaje, igual que antes. El chequeo de
conflicto que antes bloqueaba abrir CUALQUIER sesión si había una sola
viva en todo Jin ahora es **por workspace**: varios proyectos pueden
tener su pod corriendo a la vez (hasta `TERMINAL_MAX_CONCURRENT`).

Dos acciones nuevas, **solo audit, sin aprobación** — igual que ya era
`stopTerminalSession`, porque ninguna abre egress nuevo, solo destruyen
datos que ya son del owner:

- **Detener el pod** (`stopTerminalSession`): el disco se conserva.
- **Borrar el workspace** (`deleteTerminalWorkspace`, nueva): pod (si lo
  hay) + disco. Irreversible. Es la única acción de esta ampliación que
  de verdad destruye algo sin vuelta atrás.

### El reaper de inactividad solo toca pods, nunca discos

`TerminalReaperService` sigue liberando pods vencidos o inactivos (mismo
mecanismo de antes), pero ahora explícitamente **nunca** borra un PVC —
eso solo pasa por `deleteWorkspace`, una acción del owner.

### 4 bugs reales, encontrados con un K3s real (testcontainers), no con mocks

El Executor ya tenía tests de integración contra un K3s real
(`rancher/k3s` vía testcontainers) desde la tanda original; en esta
ampliación encontraron 4 bugs que los mocks no hubieran mostrado nunca,
todos sobre el ciclo de vida real de los objetos de Kubernetes (no sobre
lógica de negocio):

1. Un pod al que ya se le pidió borrarse sigue reportando `phase: Running`
   durante su `terminationGracePeriodSeconds` — `exec()` seguía
   funcionando contra un pod que ya se había pedido destruir. Fix:
   clasificarlo como `'failed'` en cuanto tiene `deletionTimestamp`, antes
   de mirar `phase`.
2. Un PVC recién borrado sigue listado hasta que su finalizer de
   protección se libera (el pod que lo usaba tiene que terminar de irse
   primero). Fix: `list()` filtra PVCs con `deletionTimestamp`, igual que
   ya hacía con pods.
3. Consecuencia directa del fix #1: al reclasificar un pod terminando
   como `'failed'`, el flujo de reanudar intentaba recrear el pod con el
   mismo nombre mientras el anterior seguía "Terminating" de verdad en
   etcd → 409 `AlreadyExists`. Fix: esperar (con tope) a que el pod
   desaparezca de verdad antes de recrear.
4. `createdAt` no era estable: al crear un workspace se guardaba con el
   reloj propio (milisegundos); al reanudarlo se leía del
   `creationTimestamp` real de Kubernetes (sin milisegundos) — el mismo
   disco reportaba dos `createdAt` distintos según cuándo se preguntara.
   Fix: siempre se usa el `creationTimestamp` real que devuelve la API al
   crear el PVC, nunca el reloj local.

### Consecuencias

- **Jin_Infra:** sin cambios de manifest más allá de lo que ya existía
  (el `StorageClass local-path` ya es el default del clúster). El tope
  real de espacio de los PVCs sigue viviendo en la `ResourceQuota` de
  `agents-sandbox` (defensa en profundidad, el owner ya lo ve en la app).
- **Jin_Executor:** `/terminal/sessions` → `/terminal/workspaces`;
  `TerminalService` → `TerminalWorkspaceService`.
- **Jin_Core:** `/api/terminal/sessions` → `/api/terminal/workspaces/
  {workspaceId}/...`; `OwnerTerminalService.requestSession()` →
  `requestStart(workspaceId, ...)`.
- **Jin_iOS:** `TerminalStore` deja de modelar "una sesión global" y pasa
  a estado por workspace (transcript, historial, servidores, aprobaciones
  pendientes, todo keyed por el `UUID` del proyecto); `TerminalView` deja
  de aceptar un proyecto opcional — cada terminal es siempre la de un
  proyecto concreto.

## Fuera de esta ampliación

- Un límite de espacio por owner visible en la app (hoy solo se ve por
  proyecto individual, vía la `ResourceQuota`).
- Migrar sesiones que ya existían antes de esta ampliación (no había
  ninguna corriendo al desplegar: el `emptyDir` anterior no tenía nada
  que preservar).

## Ampliación (2026-09-29): terminal interactiva, explorador de archivos y caché en el disco

Tras usar la terminal en el iPhone el owner pidió tres cosas: una terminal **interactiva** (`npm create vite@latest` pregunta el nombre del proyecto; el modo de un comando por vez no puede), poder **ver y editar los archivos del pod**, y que la terminal **no se cayera**. Además, un incidente reveló dos errores de despliegue nuestros (abajo).

### Terminal interactiva (PTY)

```
iPhone (SwiftTerm) ⇄ socket.io /terminal ⇄ Jin_Core ⇄ HTTP ⇄ Jin_Executor ⇄ K8s pods/exec (tty:true) ⇄ sh en /workspace
```

- **Executor:** solo transporta bytes (`TerminalPtyService`, `K8sService.openPty`), una sesión por workspace, con tamaño en vivo (resize), topes (64 KB por mensaje de teclado, 15 min sin actividad, cierre si el consumidor no da abasto) y anotación de actividad para que el reaper no libere un pod en uso.
- **Core:** dueño del audit, la auth (JWT como `/chat`), los topes (512 KB/s de entrada) y la reconexión: guarda 256 KB de salida y mantiene la sesión **10 minutos** sin app conectada; `pty:open` sobre un workspace con sesión viva se engancha en vez de abrir otra. iOS se suspende seguido; sin esto un `npm install` largo moría al minimizar.
- **iOS:** SwiftTerm como emulador. **Es la única dependencia de terceros de la app** (excepción a AGENTS.md §1.1 aprobada por el owner; solo en `JinUI`, versión exacta 1.11.2, sin shaders de Metal ni plugin de compilación). La salida del pod es texto **no confiable**: el delegado ignora portapapeles (OSC 52), enlaces (OSC 8), título e iTerm (OSC 1337), con tests.
- **Audit (lo que cambió respecto de "cada comando se audita antes de ejecutarse").** Con un TTY no hay comandos, hay teclas. `PtyInputGate` reconstruye la línea que teclea el owner (borrar, Ctrl+U/W/C, salta secuencias de escape) y Core la registra (`runTerminalCommand`, mismo actor y formato) **antes de reenviar su Enter**; si el registro falla, el Enter no llega al shell y la línea se cancela con Ctrl+C. Abrir y cerrar la sesión se auditan (`openTerminalPty`, `closeTerminalPty`). **Limitación aceptada:** dentro de un programa a pantalla completa o de un asistente, las líneas se registran igual pero sin saber a qué pregunta responden; Tab (autocompletar) y el historial con flechas no se reflejan en la línea auditada.
- **Aprobación:** sin cambios. Abrir el pod ya exige `confirm` fijo; la terminal interactiva es un canal dentro de ese pod. Las tools nuevas son virtuales (el modelo no las ve).

### Explorador y editor de archivos del pod

- Rutas `fs/list`, `fs/file`, `fs/dir`, `fs/entry` sobre el disco real del proyecto, **un archivo de texto a la vez (≤ 512 KB)**. `FS_SCRIPT` corre dentro del pod y valida cada ruta con `realpath` (ni `..` ni una carpeta que sea enlace hacia afuera pueden salir de `/workspace`); nunca sigue enlaces simbólicos; solo UTF-8; escritura atómica que conserva el modo.
- **Conflictos:** guardar manda el `sha256` que se leyó; si el archivo cambió en el pod (un comando, Claude Code) el servidor responde 409 y la app ofrece recargar, sobrescribir o seguir editando, sin perder lo tecleado.
- **Audit:** guardar, crear carpeta y borrar se auditan **antes** (fail-closed) con el hash de la **ruta**, nunca el contenido. Leer y listar no.
- Se conserva "Traer archivos al proyecto" (copia hasta 50 archivos / 256 KB al editor local).

### Dos errores de despliegue nuestros (para no repetirlos)

1. **RBAC olvidado.** Los workspaces persistentes (PR del 2026-09-28) usan PVCs y anotan actividad en el pod, pero el `Role` del Executor no tenía `persistentvolumeclaims` ni `patch` sobre `pods`: la terminal daba 502 y el reaper podía liberar un pod en uso. Cada verbo nuevo de la API de Kubernetes que use el Executor tiene que ir en `Jin_Infra/k8s/base/executor/role.yaml` **en el mismo cambio**.
2. **Caché de npm en un `emptyDir` de 512 Mi.** `npm install` de Vite lo llenaba y el kubelet **expulsaba el pod** ("EmptyDir volume tmp exceeds the limit"). Ahora `npm_config_cache` y `XDG_CACHE_HOME` viven en el PVC del proyecto (sobreviven al pod y aceleran reinstalls; `.cache` no se exporta al editor).

### Bugs que encontraron los tests con directorios y K3s reales

- `process.exit()` antes de vaciar el pipe **cortaba la salida en 64 KB**: leer un archivo grande daba JSON truncado.
- Una carrera al reservar el workspace: dos aperturas casi simultáneas pasaban las dos (en Executor y en Core).
- Un test que leía el portapapeles colgaba la suite de iOS (iOS pide permiso para leerlo).

### Consecuencias

- **Jin_Executor:** módulo `terminal` con PTY, `FS_SCRIPT` y las nuevas rutas; `K8sService.openPty`.
- **Jin_Core:** gateway `/terminal`, `TerminalPtyService`, `PtyInputGate`, rutas `fs/*` y su audit.
- **Jin_iOS:** pantalla de terminal con SwiftTerm, "Archivos del pod", `TerminalPtyStore`, `PodFilesStore`.
- **Jin_Infra:** `Role` con `persistentvolumeclaims` y `patch` sobre `pods`.

### Fuera de esta ampliación

- Varias pestañas de terminal por workspace (una PTY por workspace).
- Que el modelo use la terminal.
- Subir/bajar archivos binarios o mayores de 512 KB desde el explorador; renombrar/mover; borrado recursivo.

