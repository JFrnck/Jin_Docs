# ADR 0018 — Demos con backend, base de datos elegible y vida de hasta 7 días

Fecha: 2026-10-02. Amplía el [ADR 0006](0006-preview-services.md) (pods de servicio / previews). Decidido por el owner.

## Contexto

Los pods de preview (`startPreviewService`) se usan para **mostrar demos a clientes**. Tres límites ya costaron algo real (la app `sistema-reservas` iba a borrarse con su base de datos a las 24 h y hubo que parchearla a mano):

1. Todo vivía en un `emptyDir` (código y datos se pierden con el pod).
2. Tope duro de **24 h** y ninguna forma legítima de renovar (se editó la anotación `jin.io/expires-at` a mano).
3. **Sin internet ni `npm install`**: un backend solo podía usar módulos de Node (`node:sqlite`); no `pg`, `ioredis` ni `mongodb`.

## Decisiones

### Vida de hasta 7 días, renovable con aprobación
`PREVIEW_SERVICE_MAX_TTL_SECONDS` admite hasta 7 días **contados desde que se creó el pod**. `POST /services/:id/extend` (tool `extendPreviewService`, `confirm`) suma tiempo sin pasar de ese tope; un servicio vencido no se renueva (404) y en el tope da 409. Si K8s falla al anotar, el error sale (`replacePodAnnotationStrict`): no se miente con una fecha nueva.

### Backend con dependencias npm por Verdaccio (`template: "node"`)
Un backend Node (frontend y API en el mismo servidor, un solo puerto) instala sus dependencias por el **mismo Verdaccio** de las terminales. Mínimo privilegio, tres capas:
- label `jin.io/npm=enabled` solo si el pedido aprobado lo usa (Verdaccio solo acepta pods de servicio con él);
- NetworkPolicy propia `{serviceId}-egress` con **solo** la salida a Verdaccio, creada antes que el pod y borrada en `stop`;
- **`npm_config_ignore_scripts=true` siempre**: ningún `preinstall`/`postinstall` corre en un pod expuesto a internet (costo aceptado: dependencias con binarios nativos que lo necesiten no instalan).
El comando de arranque es **fijo** (`npm ci`/`npm install` y `npm start`); `command`/`port` del modelo se ignoran con la plantilla.

### Base de datos de demo
`db: sqlite | redis | postgres | mongodb` (datos de **prueba**, no producción):
- `sqlite`: archivo (`SQLITE_PATH`, `node:sqlite`), sin contenedor.
- `redis`/`postgres`/`mongodb`: **contenedor auxiliar nativo** (sidecar de K8s ≥ 1.29: `initContainers` con `restartPolicy: Always`) dentro del mismo pod; la app arranca cuando su `startupProbe` pasa. Solo `127.0.0.1`, **contraseña aleatoria por demo**, datos en un `emptyDir` de 512 Mi (viven lo que viva el pod, también al renovarlo), sin root, PSA `restricted`, dentro del LimitRange. Imágenes pinneadas ARM64: `redis:7.4-alpine`, `postgres:16.4-alpine`, `mongo:7.0`.
- La app recibe `REDIS_URL` / `DATABASE_URL` / `MONGODB_URI`; el motor queda en `jin.io/db-engine` y sale en `GET /services` y en la tarjeta de la app iOS.

### Salida de correo declarada en el pedido
`mailEgress: true` pone el label `jin.io/mail-egress=enabled` (antes se ponía a mano y se perdía al recrear). La NetworkPolicy del proxy `mail-egress` está en Jin_Infra (ver STATUS).

## Hallazgos de la verificación (lo que los tests unitarios no habrían visto)
Se verificó contra un **K3s real** (test de integración nuevo, ARM64) y luego en producción:
- **PostgreSQL no exigía contraseña** desde `127.0.0.1`: la imagen oficial confía en esa dirección. Corregido con `POSTGRES_INITDB_ARGS=--auth-host=scram-sha-256`; el test comprueba que una contraseña mala se rechaza.
- **MongoDB: hueco de conexión.** La imagen arranca un `mongod` temporal para crear el usuario y lo reinicia con `--auth`; un sondeo que pasa en la fase temporal deja arrancar la app y segundos después la base rechaza (`ECONNREFUSED`, visto en producción). El sondeo exige ahora el proceso definitivo (`--aut[h]` en `/proc/*/cmdline`; el patrón no se reconoce a sí mismo) y además autentica. Aun así, **una app debe reintentar su conexión al arrancar** (está en la descripción de la tool).
- **Cuota:** crear una demo sin cuota libre daba un 500 opaco y podía dejar la política de salida huérfana. Ahora es un 429 con "pide / en uso / tope" y se limpia lo creado a medias.

## Límites conocidos
- **El cuello de botella es la CPU de la VM (2 vCPU), no la memoria.** Una demo con PostgreSQL o MongoDB pide 1500m de CPU en límite (app 1000m + base); con `sistema-reservas` caben 1 demo con base de datos a la vez. La cuota de `agents-sandbox` pasó a 3000m / 6 Gi (techo de ráfaga); subirla más no ayuda.
- **Sin persistencia entre pods:** si el pod se destruye (vencimiento, `stop`, recreación) la base se pierde. Es lo acordado para demos.
- Los secretos de una demo (p. ej. la clave de Brevo de `sistema-reservas`) siguen siendo un archivo dentro del pod: no hay aún secretos por demo.

## Fuera de alcance (ADR aparte, si se hace)
Integración con git (repos por proyecto, pull/push/merge desde la app) y hosting permanente de demos.
