# Runbook: activación real de Jin (Fase 7.2)

> **Alcance:** de "el clúster está arriba" (`oci-deploy-prep.md` §8 verde) a "Jin funciona de verdad con mis cuentas": login, Telegram como canal de aprobación, Google, y un smoke test end-to-end. **Prerrequisito:** `oci-deploy-prep.md` completo hasta §7.7, con `jin-core` y `executor` en `Ready`.
>
> Cada paso marca quién lo hace. **Los valores secretos los pone el owner** — un agente nunca los maneja (ver `AGENTS.md` §5.2). Todo secreto vive en Infisical, nunca en un Secret de K8s ni en el repo.

---

## 0. Estado base (antes de tocar nada)

```bash
kubectl get pods -A                                # todo Running/Completed
kubectl -n jin get deploy jin-core                 # 1/1
kubectl -n jin exec sts/postgres -- psql -U jin -d jin -Atc \
  "select count(*) from information_schema.tables where table_schema='public'"   # 14
```

> `/health/*` **no** está bajo `/api` y la IngressRoute solo expone `/api` y `/telegram/webhook`. El health público no existe a propósito; verificalo por port-forward: `kubectl -n jin port-forward svc/jin-core 3000:3000` y `curl localhost:3000/health/ready` → `{"status":"ok"}`. `503` con `postgres:"down"` o `redis:"down"` indica qué dependencia falla.

---

## 1. Login del owner (owner)

`OWNER_PASSWORD_HASH` es un hash Argon2id (Infisical, clave de Core). Generalo **en tu laptop**, no en la VM:

```bash
cd Jin_Core && pnpm run hash-password      # te pide la contraseña, imprime el hash
```
Pegá el hash en Infisical → proyecto `jin` → `prod` → `OWNER_PASSWORD_HASH`. La contraseña en claro no se guarda en ningún lado.

Verificación (el login tiene rate limit de **5 intentos / 15 min**; no lo martilles):
```bash
curl -s -X POST https://jin.jeanfranck.com/api/auth/login \
  -H 'content-type: application/json' -d '{"password":"<tu contraseña>"}' | head -c 80
```
Debe devolver `{"accessToken":"…"}` y setear la cookie `__Host-jin_session`. Un `401` con la contraseña correcta = el hash en Infisical no corresponde.

---

## 2. Telegram — el único canal de aprobación HITL (owner)

Sin esto ninguna acción `confirm`/`dual-confirm` puede aprobarse fuera del dashboard.

1. Bot creado con `@BotFather` → `TELEGRAM_BOT_TOKEN` a Infisical.
2. `TELEGRAM_OWNER_CHAT_ID`: tu chat id numérico (escribile a `@userinfobot`). Es un **entero**; solo ese chat puede operar el bot.
3. `TELEGRAM_WEBHOOK_SECRET`: **aleatorio real** (`openssl rand -hex 32`). Es la única barrera del endpoint público — el controller rechaza con 401 todo POST sin el header `X-Telegram-Bot-Api-Secret-Token` correcto.
4. **`TELEGRAM_WEBHOOK_URL`** = `https://jin.jeanfranck.com/telegram/webhook`. Es una variable **no secreta** del Deployment (`k8s/base/jin-core/deployment.yaml`), no de Infisical. Sin ella `TelegramBotService` **nunca llama a `setWebhook`** y el bot no recibe nada.
5. **Cloudflare Access:** si `jin.jeanfranck.com` está protegido por una política de Access, el path `/telegram/webhook` necesita una política **Bypass** (Telegram no puede autenticarse). Solo ese path.

Verificación (el token se pasa por variable de entorno, no lo pegues en el comando ni lo guardes en historial):
```bash
read -rs TOKEN; export TOKEN            # pegá el token, Enter
curl -s "https://api.telegram.org/bot${TOKEN}/getWebhookInfo" | python3 -m json.tool
unset TOKEN
```
Esperado: `url` = la de arriba, `pending_update_count` bajo, sin `last_error_message`. Luego, desde Telegram: `/start`, `/status`, `/tasks`, `/budget`.

---

## 2b. Puente Claude Code ↔ owner (owner, OPCIONAL — ADR 0012)

Sin esto Jin funciona igual: el puente queda apagado y ninguna función del núcleo depende de él. Hacelo cuando quieras que una sesión de Claude Code corriendo en la VM pueda avisarte y preguntarte con botones por Telegram.

**Es un bot SEPARADO del de la sección 2.** El chat de Jin es el canal de aprobaciones HITL; el de acá no tiene ninguna maquinaria de aprobación, ni siquiera por error (`Jin_Core/src/relay/relay.isolation.spec.ts`).

1. Bot nuevo con `@BotFather` (`/newbot`) → `TELEGRAM_RELAY_BOT_TOKEN` a Infisical. **Hablale una vez** desde tu Telegram — un bot no puede escribir primero.
2. `RELAY_TOKEN`: **aleatorio real** (`openssl rand -hex 32`) a Infisical. Es una credencial distinta del `TELEGRAM_WEBHOOK_SECRET` y del JWT de tu login: un token filtrado acá solo puede relayar mensajes, no aprobar nada ni leer el resto de la API.
3. Ampliá el rol `jin-core-reader` con las dos claves nuevas — **verificalo en la base** (`psql` o el cliente que uses), la UI de Infisical v0.99 falla en silencio si el rol no las tiene.
4. `jin-relay` (`Jin_Infra/scripts/relay/jin-relay`) corre **en el nodo**, no en tu Mac: `/api/relay` está bloqueado en el ingress público (`k8s/base/jin-core/ingressroute.yaml`). Copiá el `RELAY_TOKEN` a `~/.jin-relay-token` (permisos `600`) en la VM.

Verificación, desde la VM:
```bash
scripts/relay/jin-relay send "puente activo"
```
Esperado: te llega en el chat del bot nuevo (no en el de Jin) segundos después. Si responde `503`, revisá que las dos claves estén en Infisical Y en el rol del paso 3.

---

## 3. Google — Calendar y Gmail (owner)

`jin-core` **no tiene un flujo OAuth interactivo**: usa `GOOGLE_CLIENT_ID`, `GOOGLE_CLIENT_SECRET` y un `GOOGLE_REFRESH_TOKEN` que obtenés una vez, afuera.

1. Google Cloud Console → proyecto → **APIs**: habilitar *Gmail API* y *Google Calendar API*.
2. Pantalla de consentimiento (usuario **Externo**) y credencial **OAuth client ID** tipo *Web*.
3. **Scopes mínimos** (lo que el código realmente usa — listar/leer, enviar, y CRUD de eventos):
   - `https://www.googleapis.com/auth/gmail.readonly`
   - `https://www.googleapis.com/auth/gmail.send`
   - `https://www.googleapis.com/auth/calendar.events`
4. Obtené el refresh token con el *OAuth 2.0 Playground* (engranaje → "Use your own OAuth credentials") autorizando esos scopes; **"Access type: offline"**.
5. `GOOGLE_CLIENT_ID`, `GOOGLE_CLIENT_SECRET`, `GOOGLE_REFRESH_TOKEN` → Infisical.

> ⚠️ **Refresh token de 7 días.** Con la app en estado **Testing**, Google expira los refresh tokens a los 7 días (aplica a scopes sensibles como Gmail/Calendar). Para uso personal, pasá la app a **"In production"** (sin verificar está bien para un solo usuario; verás la advertencia de "app no verificada" al autorizar). Cuando rotes el token, el comando de Telegram `/google-oauth-refreshed` reinicia el contador de antigüedad del refresh (deja constancia en el audit).

Rotar cualquiera de estos valores: `docs/runbooks/rotate-secrets.md`.

---

## 4. Canvas y Modal (owner)

- `CANVAS_BASE_URL` (URL válida) y `CANVAS_API_TOKEN` (Personal Access Token): Infisical (Core).
- `MODAL_TOKEN_ID` / `MODAL_TOKEN_SECRET`: Infisical (Executor). Solo hacen falta para código **Python** en el tier remoto; el tier local (Deno) funciona sin Modal.
- Los valores deben pasar la validación de forma de `env.schema.ts` (`CANVAS_BASE_URL` → URL, `JWT_SECRET` ≥ 32 caracteres, `TELEGRAM_OWNER_CHAT_ID` → entero).

---

## 5. Smoke test end-to-end

Corré **en este orden** y no sigas si uno falla — cada uno prueba una pieza distinta.

| # | Prueba | Cómo | Esperado |
| --- | --- | --- | --- |
| 1 | Auth | §1 | `accessToken` y cookie |
| 2 | Telegram entrega | §2 `/status` | responde el bot |
| 3 | Acción `auto` | Chat: *"¿qué eventos tengo mañana?"* | responde; fila `auto` en el audit |
| 4 | **HITL `confirm`** | Chat: *"mandá un correo de prueba a <tu correo>"* | **no se envía**; aparece una aprobación pendiente en el dashboard **y** en Telegram (`/tasks`) |
| 5 | Aprobar | `/approve <id>` o el dashboard | el correo llega; el audit registra `pending` → `approved` con el mismo `request_id` |
| 6 | Rechazar | repetí 4 y `/reject <id>` | no se envía; audit `rejected` |
| 7 | Sandbox | Chat: *"ejecutá `console.log(1+1)` en TypeScript"* | devuelve `2`; el pod efímero desaparece (`kubectl -n agents-sandbox get pods` vacío) |
| 8 | Aislamiento | desde un pod del sandbox intentá alcanzar un servicio de `jin` | bloqueado (NetworkPolicy) |
| 9 | Presupuesto | `/budget` y `GET /api/budget` | números reales, kill switch inactivo |
| 10 | Backup | `kubectl -n jin create job --from=cronjob/backup-postgres smoke-backup` | Job `Complete`; objeto nuevo en R2 |
| 11 | Restore | `kubectl -n jin create job --from=cronjob/verify-restore smoke-restore` | Job `Complete` (nunca toca datos de producción) |
| 12a | Modo por defecto | Telegram `/mode` | "supervisado (HITL completo)". Si dice otra cosa, **detené todo**: el default sembrado por la migración es `supervised` |
| 12b | Cambiar a semiautomático | `/mode semi 1` | **no cambia solo**: pide DOBLE aprobación (`/approve <id>`, esperar 30 s, `/approve <id>` otra vez); recién ahí `/mode` dice semiautomático |
| 12c | Qué se relaja | en semi: *"ejecutá `console.log(1)`"* y *"mandá un correo de prueba"* | `runCode` se ejecuta y **te avisa por Telegram**; el correo **sigue pidiendo aprobación** |
| 12d | Volver al modo seguro | `/mode safe` | inmediato, sin aprobación; avisa el cambio |
| 12e | Caducidad | `/mode semi 1` aprobado, esperar 1 h (o `UPDATE autonomy_mode_state SET expires_at = now()` y esperar 1 min) | vuelve solo a supervisado y avisa |
| 13a | Zona horaria del pod (Fase 9.4) | `kubectl -n jin exec deploy/jin-core -- date` | hora de **Lima** (UTC-5), no UTC. Si sale UTC, el cron de las 00:00 corre a las 19:00 y el de las 06:00 a la 01:00 |
| 13b | Alerta matutina 06:00 (Fase 9.4) | al día siguiente a las 06:00 (Lima) | llega por Telegram el resumen de prioridades. Si la corrida de las 00:00 falló, **lo dice** ("FALLÓ" + motivo); si no corrió, dice "NO se ejecutó". `SELECT status, ran_at FROM shadowing_runs ORDER BY ran_at DESC LIMIT 3;` muestra las corridas |
| 13c | CLI con menús (Fase 9.6) | `jin inbox` con una aprobación pendiente (paso 4) | navegas con ↑↓, Enter abre el detalle, aprobar pide confirmación; «No» está resaltado por defecto |
| 12 | Cadena del audit | `GET /api/audit` (o dashboard → Audit) y `kubectl -n jin logs deploy/jin-core \| grep -i AUDIT_CHAIN` | filas encadenadas de las pruebas 3–6; **ningún** `AUDIT_CHAIN_LOCKED` (el lock bloquea toda escritura tras detectar corrupción) |

El paso **4/5** es el que importa: es la regla de oro #7 en acción. Si el correo se envía sin pasar por aprobación, **detené todo y revertí**: es un fallo de seguridad, no un bug.

---

## 6. Después de la activación

- Anotá la activación en `Jin_Docs/STATUS.md`.
- **§3.5 de `oci-deploy-prep.md`:** decidí si rotás la llave SSH/sesión con sudo usada para el bootstrap.
- Creá un presupuesto de gasto en OCI (*Billing & Cost Management → Budgets*, umbral bajo, aviso por correo) y revisá que el almacenamiento gratuito de tu cuenta cubra los 200 GB del boot volume.
- El criterio final del BLUEPRINT (**7 días autónomo**) empieza a contar acá.

## 7. Si algo sale mal

| Síntoma | Causa probable |
| --- | --- |
| `jin-core` no llega a `Ready` | Infisical: `INFISICAL_PROJECT_ID` sigue en `PENDIENTE_…`, identidad sin acceso, o falta una clave (`kubectl -n jin logs deploy/jin-core` dice cuál) |
| Telegram no responde | Falta `TELEGRAM_WEBHOOK_URL` en el Deployment; Access sin *Bypass*; `getWebhookInfo` muestra `last_error_message` |
| Login `401` con la contraseña correcta | `OWNER_PASSWORD_HASH` de Infisical no es el hash de esa contraseña |
| Google `invalid_grant` | Refresh token expirado (app en Testing, 7 días) o revocado |
| Pod del sandbox `Forbidden` | Ver `Jin_Executor #12` (seccomp/PSA) — no debería ocurrir con la imagen actual |
| Todo caído tras un cambio | `docs/runbooks/restore-from-backup.md` / `rebuild-vm-from-scratch.md` |
