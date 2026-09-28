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
- **No es una terminal interactiva (PTY).** No hay `vim`, `top` ni programas que pidan teclas. Es un ejecutor de comandos con salida en vivo: cubre `npm create vite`, `npm install`, `npm run build`, `ls`, `cat`, `node`. Es mucho más chica y segura que un PTY, y en un teléfono es más usable.

### La única salida de red es un proxy de npm dentro del clúster

- Un **Verdaccio** propio (namespace `registry-proxy`) que solo habla con el registro de npm. Los pods de terminal solo pueden salir a ese servicio (una `NetworkPolicy` por sesión, creada por el Executor) y a DNS. Ningún otro destino de internet.
- El proxy no acepta publicar ni registrar usuarios.
- Un `postinstall` de un paquete corre dentro del pod aislado y solo alcanza el proxy.
- **Riesgo residual, aceptado:** el proxy es un canal de exfiltración de ancho de banda mínimo (pedir un paquete cuyo nombre codifica datos). Lo que hay en el pod es el proyecto del propio owner, sin secretos: los pods no reciben variables de entorno ni tokens.
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

- Terminal interactiva con PTY.
- Que el modelo use la terminal.
- Varias sesiones a la vez.
- Vite en modo desarrollo (`vite dev` con recarga): servir un servidor de desarrollo detrás del ingress necesita `allowedHosts` y HMR por WebSocket.
