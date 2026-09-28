# ADR 0015 — Editor de código en el iPhone y "Publicar" que no depende del modelo

## Contexto

Lo más importante de Jin, según el owner (2026-09-28), es **levantar entornos (pods) y servir links dinámicos**. Hasta ahora eso solo ocurría si el **modelo** llamaba a `startPreviewService`. Cuando el proveedor rechazaba el pedido, el owner no podía trabajar.

- El rechazo se reproduce (el pedido original se corta con un refusal del proveedor). #50 lo hace visible y registra `stopReason`/`modelId`, pero la causa exacta **sigue sin confirmarse**: el owner todavía no reintentó con #50 desplegado.
- `PreviewServicesController` solo tenía `GET` y `DELETE`: ningún camino para que el **owner** levante un pod.
- El "Editor de código" de la web es una demo (Monaco con un `parser.ts` fijo). En la app decía "solo en escritorio".

## Decisiones

### Un camino directo del owner, sin modelo de por medio

`POST /api/preview-services` (JWT del owner) crea el mismo pedido `startPreviewService`. El modelo pasa a ser un ayudante opcional, no una puerta obligatoria.

### No es una excepción a HITL

- La decisión del nivel sigue siendo `HitlPolicyService.decide` (nivel del registry → overrides de feature flags → modo de autonomía), la misma puerta que usa `AgentService`. Cambia solo *quién* inicia.
- Con `confirm`: aprobación pendiente con `actor: 'owner:api'`. Telegram, la web, la app y push la ven igual. Responde `pending-approval`.
- Si un modo de autonomía la relajó a `notify`: se ejecuta, deja su fila en el audit y avisa post-hoc, como cualquier acción relajada. Responde `started`.
- El owner eligió esto ante la alternativa de saltarse la aprobación: publicar un link público es lo que HITL existe para vigilar.

### Acotado

- Máximo 50 archivos y 256 KB en total, rutas sin `..` ni `/` inicial, TTL de 60 s a 24 h, sin campos extra.
- Con `template: "static"` hace falta `index.html`; se valida **antes** de crear la aprobación, para no dejar una pendiente que haya que rechazar.
- Express cortaba el JSON en 100 KB, así que un proyecto de 100–256 KB ni llegaba al validador. El techo del parser sube a 512 KB (el tope real es Zod) y los 4xx del parser (413, JSON roto) dejan de salir como 500.

### El editor es nativo y sin dependencias

- `UITextView` con TextKit, sin librerías. Selección, cursor y scroll son los del sistema, que era la razón del pedido ("menos toques accidentales" en una app nativa).
- Resaltado con un tokenizador propio (HTML, CSS, JS/TS/JSX, JSON, Markdown): función pura y testeada, lineal, que no se rompe con código incompleto. HTML resalta `<script>` como JS y `<style>` como CSS.
- Autoindentado, pares automáticos, cerrar etiquetas HTML al escribir `>`, y una barra de teclas sobre el teclado con los símbolos que el iPhone esconde.
- No hay autocompletado, y la pantalla no lo promete.
- Publicar está en la barra superior, lejos del área de escritura, y abre una hoja con el resumen antes de enviar. Borrar un archivo, una carpeta o un proyecto pide confirmación; no hay deslizar para borrar.
- La barra de pestañas se oculta dentro del editor.

### Los proyectos viven en el iPhone

`code-projects.json` en Application Support, con protección de archivo (`.completeFileProtection`) y guardado agrupado (debounce). Las carpetas se deducen de las rutas; las vacías solo existen en el teléfono (el servidor recibe archivos).

### Estado de lo publicado

Al publicar queda un registro por proyecto (pendiente / publicada / no aprobada). Como el servidor no devuelve el pod al aprobar, la app lo reconoce en la lista de apps por el prefijo del slug (`<nombre>-<sufijo>`) y el TTL, y solo concluye "no se aprobó" si las dos listas se cargaron bien.

### Plantillas

React + Tailwind, HTML/CSS/JS, Vue y en blanco. Todas estáticas: el pod **no tiene internet**, así que las librerías las carga el navegador del owner desde CDN.

## Sobre los "lineamientos"

- **Del modelo:** este cambio quita la dependencia. Publicar ya no exige que el modelo coopere. La causa real del rechazo se confirma cuando el owner reintente y se mire `stopReason` en Loki. **No** se enruta automáticamente un rechazo a otro proveedor: sería sortear una barrera de seguridad en lugar de diagnosticarla; si es un falso positivo, se reporta al proveedor con el `stopReason` en mano.
- **De Apple:** el código no se ejecuta en el dispositivo. Se edita ahí y corre en pods del servidor; el link se abre en `SFSafariViewController`. Es el modelo de los IDE en la nube. La regla 2.5.2 prohíbe descargar y ejecutar código **en el dispositivo** que cambie la app.

## Lo que no está (a propósito)

- **Proyectos con build (Vite, npm).** No se pueden publicar hoy: `npm install` y `vite build` necesitan internet y el pod es default-deny. Pide un paso de build en el servidor, con salida solo al registro de npm. Ese permiso de red es una decisión de seguridad del owner.
- **Terminal.** Ejecutar comandos arbitrarios en un pod con aprobación por comando es lo más delicado. Va después del paso de build, y con su propia ADR.
- Que Jin edite el proyecto por sí solo, guardar proyectos en el servidor, republicar sobre el mismo pod y autocompletado.

## Consecuencias

- **Jin_Core:** `OwnerPreviewPublishService`, un endpoint y su contrato OpenAPI (solo agrega). Sin migración.
- **Jin_iOS:** `CodeProject`, `ProjectsStore`, `SyntaxHighlighter`, `EditorRules`, el editor y sus pantallas.
- **Verificado:** el ciclo completo contra un Jin_Core local con un executor falso (publicar → aprobación pendiente `owner:api` → aprobar en la app → "Publicada" con link y cuenta regresiva), y el editor en el simulador. **No verificado:** escribir con el teclado real del iPhone y abrir un pod real desde la app.
