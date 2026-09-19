# Runbook: VM perdida → reconstrucción desde IaC + backups

## Para quién es esto

Procedimiento operativo (`BLUEPRINT.md` línea ~637, `ANALISIS.md`: "riesgo residual real: Oracle es conocido por reclamar instancias Always Free con poco uso... ten el runbook de 'VM perdida' escrito") para el escenario en el que la VM de OCI desaparece por completo (reclamada por Oracle, borrada por error, hardware perdido) — no un simple restart, la VM ya no existe.

**La garantía de fondo:** todo en `Jin_Infra` es IaC (manifests + scripts de bootstrap), y los datos reales (Postgres/Redis/`memory.db`) están respaldados en R2, independiente de la VM. Perder la VM es reconstruible en horas, no un evento catastrófico — siempre que los backups en R2 sigan ahí.

## 1. Provisionar la VM nueva

Misma spec que la original (`docs/runbooks/oci-deploy-prep.md` §1: ARM Ampere, 2 vCPU / 12 GB RAM / 200 GB, Ubuntu 24.04 LTS, Always Free). Si la cuenta de OCI también se perdió, esto ya no es un runbook de este repo — es recuperación de cuenta con Oracle.

## 2. Correr el runbook de deploy completo, con una diferencia

Seguí `docs/runbooks/oci-deploy-prep.md` de punta a punta (hardening del SO, K3s, `02-seed-secrets.sh`, manifests, Flux, `08-seed-infisical-app-secrets.sh`) — es el mismo procedimiento que un deploy nuevo, porque técnicamente lo es: no queda nada de la VM vieja.

**La diferencia real está en el §8 (verificación final) de ese runbook:** en un deploy nuevo, Postgres/Redis/`memory.db` arrancan vacíos y eso es correcto. Acá, **antes de considerar la reconstrucción terminada**, hay que restaurar los datos reales:

1. Completá `oci-deploy-prep.md` hasta que `jin-core`/`executor` estén `Ready` con datos vacíos (confirma que la infra en sí está sana).
2. Seguí `docs/runbooks/restore-from-backup.md` completo (Postgres + `memory.db` — Redis normalmente no vale la pena, ver ese runbook §2) contra los backups más recientes en R2.
3. Corré `scripts/backup/verify-restore.sh` una vez más contra el clúster nuevo, para confirmar que los backups en R2 siguen siendo válidos y que el ciclo completo (backup viejo → clúster nuevo) funciona de punta a punta — es la prueba real de que la garantía de "reconstruible desde IaC + backups" es cierta, no solo teórica.

## 3. Qué NO se recupera solo con IaC + backups

- **Las credenciales de la identidad de máquina de Infisical** (`INFISICAL_CLIENT_ID`/`SECRET` de `jin-core`/`jin-executor`) — Infisical mismo arranca vacío en la VM nueva (es un servicio corriendo en el clúster, no algo respaldado en R2 hoy). Hay que recrear el proyecto + las 2 identidades desde cero (§7.6 de `oci-deploy-prep.md`) y volver a cargar ahí las 15 claves reales.
- **El túnel de Cloudflare y los CNAMEs wildcard** — son configuración externa al clúster (dashboard de Cloudflare), sobreviven a la pérdida de la VM sin cambios, pero el token del túnel nuevo si se regeneró sí hay que re-sembrarlo.
- **Cualquier estado que viviera solo en Redis** — ver `restore-from-backup.md` §2, no es información que importe conservar.

## 4. Después de reconstruir

Documentá qué pasó con la VM vieja (por qué se perdió) fuera de este repo — si fue por inactividad de OCI Always Free, es una lección operativa (mantener actividad mínima), no algo que este runbook pueda prevenir por sí solo.
