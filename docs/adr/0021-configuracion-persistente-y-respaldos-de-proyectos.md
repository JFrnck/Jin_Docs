# ADR 0021 — Configuración de publicación que permanece y respaldos de proyectos

Fecha: 2026-10-05. Amplía el [ADR 0015](0015-editor-y-publicar-sin-modelo.md) (editor y publicar) y el [ADR 0020](0020-variables-de-entorno-por-demo.md) (variables por demo). Decidido por el owner.

## Contexto

El owner escribía las claves (Brevo) como archivos del proyecto del editor y **nunca se aplicaron ni permanecieron**; las variables nuevas (ADR 0020) había que reescribirlas en cada publicación; y no había dónde ver ni restaurar un proyecto terminado. Causas halladas:

1. `publishFiles` mandaba **todos** los archivos y la plantilla estática servía todo menos `.jin/`: un `/.env` con una clave real de Brevo estuvo **descargable** (2 veces; la clave se rotó). Además nada lee un `.env` como variables de entorno.
2. La hoja Publicar arrancaba vacía: los valores no se recordaban (por diseño, ADR 0020).
3. La plantilla `static` **pisa `.jin/static-server.mjs`** con el servidor de solo-GET de Jin, así que un backend propio no arranca; y Core rechaza `db` con `static`.
4. `sistema-reservas` no es lo normal: las apps del owner son React/Tailwind/Next/Nest. **Node con npm es el caso normal; HTML estático, la excepción.**
5. Los datos de una demo viven en un `emptyDir`: detener el pod los pierde. (Sin resolver aquí: fase 3.)

## Decisiones

### Configuración por proyecto, valores en el Llavero
`ProjectPublishConfig` vive **dentro del proyecto** (`code-projects.json`): tipo (Node npm / HTML estático), base de datos, correo, TTL y **nombres** de variables. Los **valores** se recuerdan solo en el **Keychain del iPhone** (`WhenUnlockedThisDeviceOnly`, sin iCloud), por proyecto; se rellenan solos al publicar, hay "Olvidar los valores guardados" y se borran con el proyecto. Siguen sin ir a Postgres, audit, respaldos ni logs; solo viajan en la petición HTTPS de publicar (ADR 0020).

### Node (npm) por defecto
Sin configuración guardada, el tipo se infiere: Node salvo un proyecto sin `package.json` y con `index.html`. Toda base de datos y las variables piden Node (Core ya rechazaba `db` con `static`). El comando del template `node` pasa a `npm ci|install && npm run build --if-present && exec npm start`. Límites: pod de 1 GiB / 1 CPU y `ignore-scripts` siempre activo (un `next build` pesado puede quedarse corto).

### Guardia de secretos (cliente y servidor)
Archivos que parecen secretos (`.env*`, `*.pem|key|p12|pfx`, `id_rsa*`, `.npmrc`, `.netrc`, `brevo.json`; se permiten `.env.example|sample|template`) **no se publican ni se respaldan**: la app lo bloquea y Core lo rechaza (400, nombra el archivo, nunca repite su contenido) para cualquier template y camino (owner y modelo). El servidor estático de Jin ya no sirve **ningún** archivo ni carpeta oculta. Con `mailEgress` el Executor inyecta `MAIL_EGRESS_PROXY` (no secreto, nombre reservado): el owner no escribe la URL del proxy.

### Respaldos de proyectos en Core
Tabla `project_snapshots` (migración 0016) y API del owner `/api/project-snapshots` (guardar, listar sin archivos, obtener con archivos, borrar): **código + configuración + nombres de variables**, jamás valores; topes de Publicar (50 archivos / 256 KB), 100 respaldos como máximo; cada acción en el audit sin el contenido. No pasa por HITL (es dato propio, sin efectos fuera de Jin, como `exportPreviewFiles`). El dump nocturno de Postgres ya lo cifra y sube a R2. En la app: **Más → Proyectos respaldados** (ver, restaurar como proyecto nuevo, borrar) y editor ••• → **Respaldar en Jin**. GitHub queda como exportación opcional posterior (Executor#32, apagado hasta que existan la GitHub App y las claves).

## Pendiente (no construido)
- **Fase 3:** respaldo/restauración de los **datos** de una demo (SQLite) y snapshot automático antes de que el reaper la borre.
- **Fase 4:** "Subir a GitHub" sobre un respaldo (requiere repo `jin-demos`, GitHub App y claves en Infisical, pasos del owner).
- Nombres de demo limpios en lugar del sufijo aleatorio (decisión del owner pendiente).

## PRs
Executor#36 · Core#71 (guardia + estático), Core#72 (respaldos, apilado sobre #71), Core#73 (build en el template node) · iOS#16 (configuración y Llavero), iOS#17 (respaldos, apilado sobre #16). Orden: Core#71 → #72 (migración 0016 antes del rollout) → #73; Executor#36; app.
