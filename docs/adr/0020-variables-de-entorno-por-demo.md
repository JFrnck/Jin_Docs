# ADR 0020 — Variables de entorno por demo, creadas desde Jin

Fecha: 2026-10-04. Amplía el [ADR 0018](0018-demos-con-backend-y-base-de-datos.md) (demos con backend). Decidido por el owner.

## Contexto

El ADR 0018 dejó `secrets: ["brevo"]`: una demo puede **consumir** un Secret compartido `demo-secret-<nombre>`, pero **crearlo** era un `kubectl` manual y no se borra con la demo. El owner pidió lo que faltaba: un sistema dentro de Jin para **crear variables de entorno por demo/pod**, que **se borren cuando se borra el pod**, sin que el valor llegue al modelo, al audit ni a la base.

## Decisiones

### Un Secret por demo con `ownerReference` al pod
El Executor crea `demo-env-<serviceId>` **antes** del pod, y cuando el pod existe le añade `ownerReference` (uid del pod). Kubernetes lo borra solo al borrarse el pod (vencimiento, `stop`, reaper); además `stop()` lo borra explícito. Si falla crear el pod o fijar el dueño, la limpieza existente (`stop()`) borra todo: nada huérfano. El pod las recibe por `envFrom` (`optional: false`); el valor **no** aparece en el spec del pod (`kubectl get pod -o yaml` no lo muestra). La anotación `jin.io/env-names` guarda **solo los nombres** y `GET /services` los devuelve.

### Permiso mínimo del Executor (RBAC, autorizado por el owner)
Role `executor-agents-sandbox`: `secrets` → `create`, `patch`, `delete` en `agents-sandbox`. **Sin `get`/`list`**: el Executor escribe el valor pero no puede leerlo de vuelta ni leer otros Secrets (Jin_Infra#58).

### El valor nunca toca al modelo
Se escribe en la hoja **Publicar** de la app iOS (`SecureField`) y viaja por HTTPS autenticado a `POST /api/preview-services`. El chat nunca lo ve ni lo pide; un `env` dentro del payload de una tool del modelo se **ignora**.
- **Core guarda los valores solo en RAM** (`EnvVaultService`: por `requestId`, 24 h, tope 50, `take()` lee y borra, `toJSON` vacío, oculto a `inspect`). Nunca en Postgres (el `payload` de la aprobación es `jsonb`), ni en el audit, ni en logs.
- El `payload`, el hash, el `planSummary` y el audit llevan **solo nombres** ("con 2 variables de entorno: A, B; valores ocultos").
- Si Core se reinicia con una aprobación pendiente, esa publicación falla con un mensaje claro ("reenvía las variables"): costo aceptado a cambio de no persistir secretos.
- El teléfono no guarda los valores (ni proyecto, ni `UserDefaults`, ni Llavero); `description`/`debugDescription`/`dump` de los tipos que los contienen los ocultan.

### Reglas (idénticas en Executor, Core e iOS)
Nombre `^[A-Z][A-Z0-9_]{0,63}$`; ≤ 30 variables; valor ≤ 8 KB; total ≤ 64 KB. Reservadas: `PORT PATH HOME HOSTNAME USER SHELL TZ CI PNPM_HOME XDG_CACHE_HOME DATABASE_URL REDIS_URL MONGODB_URI SQLITE_PATH HTTP_PROXY HTTPS_PROXY NO_PROXY ALL_PROXY` y los prefijos `NODE_ LD_ NPM_CONFIG_ KUBERNETES_`. Los errores nunca repiten un valor.

### Se mantiene el Secret compartido
`secrets: ["brevo"]` + `demo-secret-<n>` sigue valiendo para credenciales reutilizables entre demos (lista blanca `PREVIEW_SERVICE_ALLOWED_CREDENTIALS`).

## Verificación
- Unit en los tres repos (validación, orden Secret→Pod→dueño, limpieza ante fallos, ausencia de valores en logs/respuesta/pod spec/audit/payload).
- Integración K3s real (Executor): la variable llega al contenedor, el valor no está en el pod spec, el `ownerReference` coincide con el uid y **el Secret desaparece al borrar el pod**.

## Fuera de alcance / pendiente
- Editar variables de un pod **ya corriendo** (las variables de un contenedor son inmutables: se recrea la demo y se pierde su disco).
- Que Jin las **pida desde el chat** sin ver el valor (`requiredEnv`: solo nombres; la app mostraría el formulario en la aprobación pendiente): no construido.
- Rotación automática; guardar valores en Postgres/Infisical; mostrar un valor de vuelta en la app.
- Nombres de demo limpios (`reservas.jinserver.com`) en lugar del sufijo aleatorio: pendiente de decisión del owner (el sufijo es hoy la única barrera contra adivinar la URL, ADR 0006 punto 9).

## Orden de despliegue
Infra#58 (RBAC; requiere autorización explícita) → Executor#35 → Core#70 → app iOS#14.
