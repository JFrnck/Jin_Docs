# Runbook: restore real desde backup

## Para quién es esto

Procedimiento operativo (`AGENTS.md` línea 394) para restaurar Postgres, Redis o `memory.db` desde un backup real de R2 tras pérdida/corrupción de datos — a diferencia de `Jin_Infra/scripts/backup/verify-restore.sh` (que restaura a un scratch/`restore_test` descartable, corre mensual, y **nunca toca los datos reales**), esto reemplaza los datos en producción. **Es destructivo — no lo corras sin estar seguro de que el dato actual está perdido o es peor que el del backup.**

**Antes que nada:** `verify-restore.sh` ya prueba que los backups son restaurables (§3 del blueprint: "ningún backup no probado cuenta como backup") — este runbook reusa exactamente la misma mecánica de descarga+descifrado, solo que apunta al destino real en vez de a un scratch.

## 0. Decidí el alcance

¿Se perdió Postgres completo, Redis, `memory.db`, o los tres? Restaurá solo lo que hace falta — restaurar de más arriesga perder escritura reciente en lo que sí estaba sano.

## 1. Postgres

```bash
# Bajar jin-core primero -- no debe escribir mientras restauramos.
kubectl -n jin scale deployment/jin-core --replicas=0

# Mismo mecanismo de descarga+descifrado que verify-restore.sh (ver ese
# script para list_r2_backups/age_decrypt_file), apuntando al backup real
# que quieras restaurar (no necesariamente el más reciente si la
# corrupción es previa).
rclone --config "${RCLONE_CONFIG}" copyto "r2:${R2_BUCKET}/<ruta-del-backup>" ./postgres.age
age --decrypt --identity "${AGE_IDENTITY_PATH}" --output ./postgres.dump ./postgres.age

# Restore real -- sobre la base "jin", no "restore_test".
kubectl -n jin cp ./postgres.dump postgres-0:/tmp/postgres.dump
kubectl -n jin exec -i statefulset/postgres -- env PGPASSWORD="${POSTGRES_PASSWORD}" \
  pg_restore -U jin -d jin --clean --if-exists --no-owner --no-privileges /tmp/postgres.dump

# Verificación mínima antes de levantar jin-core de nuevo.
kubectl -n jin exec -i statefulset/postgres -- env PGPASSWORD="${POSTGRES_PASSWORD}" \
  psql -U jin -d jin -Atc "SELECT count(*) FROM audit_log;"

kubectl -n jin scale deployment/jin-core --replicas=1
kubectl -n jin rollout status deployment/jin-core
```

## 2. Redis

Redis en este proyecto solo guarda contadores de rate-limiting (`Jin_Core/src/rate-limit/redis-throttler-storage.service.ts`) — **no hay estado que perder que importe de verdad** (a diferencia de Postgres). Si Redis se corrompe, lo más simple casi siempre es **no restaurar y dejar que arranque vacío** (`FLUSHALL` implícito al perder el PVC) — los contadores se reconstruyen solos con el tráfico normal.

Restaurar desde backup solo si por algún motivo hace falta preservar el estado exacto:

```bash
rclone --config "${RCLONE_CONFIG}" copyto "r2:${R2_BUCKET}/<ruta-del-backup>" ./redis.age
age --decrypt --identity "${AGE_IDENTITY_PATH}" --output ./dump.rdb ./redis.age

kubectl -n jin scale statefulset/redis --replicas=0
kubectl -n jin cp ./dump.rdb redis-0:/data/dump.rdb
kubectl -n jin scale statefulset/redis --replicas=1
```

## 3. `memory.db` (memoria del agente)

```bash
kubectl -n jin scale deployment/jin-core --replicas=0

rclone --config "${RCLONE_CONFIG}" copyto "r2:${R2_BUCKET}/<ruta-del-backup>" ./memory.age
age --decrypt --identity "${AGE_IDENTITY_PATH}" --output ./memory.db ./memory.age

# El PVC jin-core-data monta en /data/memory (ver k8s/base/jin-core/deployment.yaml)
# -- necesita un pod temporal con el mismo PVC montado para copiar el archivo,
# porque jin-core está en 0 réplicas.
kubectl -n jin run restore-helper --rm -it --image=busybox --restart=Never \
  --overrides='{"spec":{"containers":[{"name":"restore-helper","image":"busybox","command":["sleep","300"],"volumeMounts":[{"name":"data","mountPath":"/data"}]}],"volumes":[{"name":"data","persistentVolumeClaim":{"claimName":"jin-core-data"}}]}}' &
sleep 5
kubectl -n jin cp ./memory.db restore-helper:/data/memory/memory.db
kubectl -n jin delete pod restore-helper --grace-period=0 --force

kubectl -n jin scale deployment/jin-core --replicas=1
kubectl -n jin rollout status deployment/jin-core
```

## 4. Después de cualquier restore

- Corré `verify-restore.sh` igual (aunque acabes de restaurar a mano) — confirma que lo que quedó en producción también pasa las mismas verificaciones de integridad que corren mensualmente.
- Documentá qué se perdió, desde qué backup se restauró, y la ventana de datos perdidos (todo lo escrito entre el backup usado y el momento de la falla) — información real para el owner, no para este repo.
