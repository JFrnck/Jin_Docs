# ADR 0022 — GitHub desde la app: clonar, ramas, pull y push

Fecha: 2026-10-06. Amplía el [ADR 0019](0019-guardar-demos-en-github.md) (GitHub App, token efímero en el Executor) y el [ADR 0021](0021-configuracion-persistente-y-respaldos-de-proyectos.md). Decidido por el owner: alcance completo (clonar, pull, push, ramas, graduar demos), GitHub App, y que los repos vivan "en el editor" sin tope de tamaño.

## Contexto

No había integración con GitHub en la app ni forma de clonar un repo. El editor del iPhone guarda todo en un solo `code-projects.json` y publica por una petición JSON (Core 512 KB, Executor 1 MB, workspace en una env del pod): un repo real (React/Next/Nest) no cabe (tope 50 archivos / 256 KB). Subir ese tope a "ilimitado" rompería el guardado, la publicación y el pod. Los pods del workspace no tienen `git` ni salida a github.com (default-deny).

## Decisiones

### El repo vive en el disco de la terminal; el editor lo abre en vivo
Un proyecto "de GitHub" (`CodeProject.github`) guarda el código en el PVC `terminal-ws-<id>` (ya existe uno por proyecto) y se edita con **Archivos del pod** (editor en vivo ya construido): sin tope de archivos; el límite real es el disco del workspace. El editor local clásico sube a 500 archivos / 5 MB para proyectos pequeños; **Publicar y Respaldar no cambian** (50 / 256 KB).

### Todo git corre en el Executor, sobre una copia temporal
`GithubReposService`: copia `/workspace/<dir>` (con `.git`, sin `node_modules` ni carpetas del entorno) a un temporal por **tar binario** (exec), corre git ahí y, si hubo cambios, devuelve el resultado al pod (extrae a un temporal del mismo disco; si tar falla no se toca nada). Clon superficial (`--depth 100`). El token de instalación (1 h) solo entra al entorno del `git` hijo (`GitRunner`); el pod nunca lo ve. Tope de 200 MB por operación.

### Repos permitidos = los de la instalación de la App
`GET /installation/repositories` reemplaza a una lista blanca por variable para clonar/operar. Sin las tres claves (`GITHUB_APP_ID`, `GITHUB_APP_INSTALLATION_ID`, `GITHUB_APP_PRIVATE_KEY`) todo está apagado (503 con mensaje claro).

### Niveles HITL por operación
- Listar repos, clonar, estado, ramas, checkout y pull (**solo fast-forward**) → del owner, `auto` + fila de audit (clone/checkout/pull), como `exportPreviewFiles`.
- **Push** → `pushGithubBranch`: `confirm` + `humanDecision` (ningún modo de autonomía lo relaja). La aprobación dice repo, nº de archivos y rama NUEVA; no se crea si no hay cambios. Sin `--force`; **nunca** a `main`/`master`/la rama por defecto; los archivos que parecen secretos (`.env*`, claves…) frenan TODO el push (misma guardia que ADR 0021).
- Cambio de HITL (revisión del owner, CLAUDE.md §2.1): `registry.ts` suma esa tool (23 tools; 3 marcas `humanDecision`).

### Flujo en la app
Más → **GitHub**: lista de repos → **Clonar** (proyecto nuevo o uno existente, carpeta y rama). Con la terminal apagada se pide abrirla (aprobación del owner) y el clon sigue solo cuando corre. En el proyecto: rama, archivos cambiados, Archivos del pod, Terminal, Actualizar, Ramas, **Subir cambios** (rama nueva + mensaje → aprobación).

## Fuera de alcance / pendiente
- **Graduar demos** (`demo/<slug>` → repo propio) y restaurar demos guardadas; **Proyectos respaldados → Subir a GitHub**.
- Pull requests, merge/rebase, resolver conflictos (se resuelven editando los archivos y repitiendo pull/push).
- Prueba contra GitHub real (necesita que el owner cree la App) y K3s real de `PodWorkspaceFs` (el script del pod se probó con `sh` real y en `node:22-alpine`).

## Pasos del owner (no los hace Jin)
Crear la GitHub App (Contents RW + Metadata R, sin webhook), instalarla en sus repos, y guardar `GITHUB_APP_ID`, `GITHUB_APP_INSTALLATION_ID`, `GITHUB_APP_PRIVATE_KEY` (y `GITHUB_DEMOS_REPO` para demos) en Infisical, con lectura para el Executor.

## PRs
Executor#37 · Core#75 · iOS#21. Orden: Executor → Core → app (más las claves en Infisical).
