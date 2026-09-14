# Runbook: rotación trimestral de secretos

## Para quién es esto

Procedimiento operativo (`AGENTS.md` línea 394) para la rotación trimestral obligatoria (`BLUEPRINT.md` §11) de todos los secretos de Jin. Pensado para el owner o un agente con acceso real a la VM — ninguno de estos pasos es automatizable a ciegas: rotar un secreto real requiere generar/copiar el valor nuevo desde el proveedor correspondiente.

## 1. Secretos que viven en Infisical (desde Fase 8.1)

Las 13 claves de Jin_Core + las 2 de Jin_Executor (ver la lista exacta y actualizada en `Jin_Core/src/config/secrets-loader.ts` y `Jin_Executor/src/config/secrets-loader.ts` — no la dupliques de memoria acá, esos archivos son la fuente de verdad).

Para cada una:
1. Generá/copiá el valor nuevo desde el proveedor real (Anthropic, Google AI Studio, Canvas, Telegram, Google Cloud Console, etc.).
2. Actualizala en Infisical (UI, o `infisical secrets set <CLAVE> <valor-nuevo> --env=prod` si tenés el CLI autenticado).
3. **No hace falta reiniciar `jin-core`/`jin-executor`** para que tomen el valor nuevo en la próxima llamada — `loadSecrets()` corre una sola vez al startup del proceso (ver `secrets-loader.ts`), así que el pod actual sigue con el valor viejo en memoria hasta el próximo restart. Si la rotación es por sospecha de compromiso (no solo calendario), forzá el restart: `kubectl -n jin rollout restart deployment/jin-core` (o `deployment/executor` en `jin-executor`).
4. Confirmá que el pod nuevo arrancó `Ready` antes de considerar la rotación cerrada — si el valor nuevo es inválido, `validateEnv()` lo va a rechazar ruidoso al arrancar (fail-fast, `AGENTS.md` 8.4).

## 2. Credenciales de la identidad de máquina de Infisical (Universal Auth)

`INFISICAL_CLIENT_ID`/`INFISICAL_CLIENT_SECRET` de `jin-core` y de `jin-executor` (2 identidades separadas, ver `Jin_Infra/scripts/bootstrap/08-seed-infisical-app-secrets.sh`) — estas SÍ viven fuera de Infisical, en el Secret de K8s `jin-core-infisical-auth`/`jin-executor-infisical-auth`.

1. En la UI de Infisical: Settings del proyecto → Identities → la identidad correspondiente → regenerar el Client Secret (esto invalida el viejo de inmediato).
2. Re-aplicar el Secret de K8s con el valor nuevo:
   ```bash
   kubectl -n jin create secret generic jin-core-infisical-auth \
     --from-literal=INFISICAL_CLIENT_ID=<sin-cambios> \
     --from-literal=INFISICAL_CLIENT_SECRET=<nuevo> \
     --dry-run=client -o yaml | kubectl apply -f -
   ```
   (mismo patrón para `jin-executor-infisical-auth` en el namespace `jin-executor`).
3. `kubectl -n jin rollout restart deployment/jin-core` (y `jin-executor` según cuál identidad rotaste) — el Secret montado vía `envFrom` no se actualiza en un pod ya corriendo.

## 3. Secretos de bootstrap (siguen en un Secret de K8s, seed en `02-seed-secrets.sh`)

`POSTGRES_PASSWORD`, `REDIS_PASSWORD`, `CLOUDFLARE_API_TOKEN`, `CLOUDFLARED_TUNNEL_TOKEN`, `GRAFANA_ADMIN_PASSWORD`, `AGE_PUBLIC_KEY`/`AGE_PRIVATE_KEY`, `R2_*`, `GHCR_USERNAME`/`GHCR_PAT`, y las credenciales de bootstrap de Infisical mismo (`INFISICAL_ENCRYPTION_KEY`/`INFISICAL_AUTH_SECRET`) — ver la cabecera de `Jin_Infra/scripts/bootstrap/02-seed-secrets.sh` para la lista completa y actualizada.

1. **`POSTGRES_PASSWORD`/`REDIS_PASSWORD`:** requieren cambiar la password real en el motor (`ALTER USER jin WITH PASSWORD '...'` para Postgres; `CONFIG SET requirepass ...` + persistir en el StatefulSet para Redis) ANTES de actualizar el Secret — si actualizás el Secret primero, `jin-core` pierde la conexión hasta que ambos coincidan.
2. **`CLOUDFLARE_API_TOKEN`/`CLOUDFLARED_TUNNEL_TOKEN`:** regenerar en el dashboard de Cloudflare (Zero Trust → Tunnels / API Tokens), luego re-aplicar el Secret y reiniciar `cloudflared`/`cert-manager` según cuál cambió.
3. **`AGE_PUBLIC_KEY`/`AGE_PRIVATE_KEY`:** rotar esta pareja es más delicado — los backups viejos quedaron cifrados con la llave vieja. Generá el par nuevo (`age-keygen`), pero **conservá la llave privada vieja** (en el llavero, no en el clúster) hasta que ningún backup cifrado con ella siga dentro de la ventana de retención (ver `Jin_Infra/scripts/backup/`).
4. Para cualquiera de estos: `apply_secret` en `02-seed-secrets.sh` es idempotente — re-correr el script entero con las variables de entorno actualizadas es válido, no hace falta editarlo.

## 4. Después de rotar cualquier secreto

- Guardá el valor nuevo en el llavero del owner (Bitwarden/1Password) — nunca solo en Infisical/K8s.
- Actualizá la fecha de "última rotación" donde el owner la lleve registrada (fuera de este repo).
- Si la rotación fue por sospecha de compromiso (no solo calendario): revisá `audit_log` por actividad inusual en la ventana de exposición antes de cerrar el incidente.
