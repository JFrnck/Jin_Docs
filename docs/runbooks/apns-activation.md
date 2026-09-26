# Activar las notificaciones push de la app iOS (APNs)

Todo está construido y desplegado **apagado** (ADR 0014). Este runbook es lo único que falta el día que pagues la cuenta Apple Developer (99 USD/año). Tiempo estimado: 20 minutos.

**Qué NO cambia al activarlo:** desde una notificación no se puede aprobar nada. Las aprobaciones llegan sin acciones y tocar abre la tarjeta. Solo Claude Code, que es mensajería, permite responder desde la notificación.

---

## 1. Cuenta y equipo (owner, en el Mac)

1. Inscríbete en el Apple Developer Program con tu Apple ID: <https://developer.apple.com/programs/enroll/>. La aprobación puede tardar desde minutos hasta 48 h.
2. En Xcode: *Settings → Accounts →* tu Apple ID. Tiene que aparecer un equipo **sin** "(Personal Team)". Anota su **Team ID** (10 caracteres, en *developer.apple.com → Membership*).

## 2. Clave de APNs (owner, en el portal)

1. <https://developer.apple.com/account/resources/authkeys/list> → **+** → marca **Apple Push Notifications service (APNs)** → *Continue* → *Register*.
2. **Descarga el `.p8`.** Apple lo deja bajar **una sola vez**. Anota el **Key ID** (10 caracteres).
3. Guarda el `.p8` en tu gestor de contraseñas. No lo subas a ningún repo ni lo pegues en un chat.

> Una sola clave sirve para sandbox (builds de Xcode) y producción (TestFlight/App Store), y no vence. Si se filtra, revócala en el mismo portal y crea otra.

## 3. Secretos en Infisical (owner)

En el proyecto de Jin, entorno **Production**, crea:

| Clave | Valor |
|---|---|
| `APNS_KEY_ID` | Key ID del paso 2 |
| `APNS_TEAM_ID` | Team ID del paso 1 |
| `APNS_PRIVATE_KEY` | contenido completo del `.p8`, con `-----BEGIN PRIVATE KEY-----` y los saltos de línea (también sirven los `\n` escapados) |

Son **las tres o ninguna**: con una a medias, jin-core no arranca y dice cuál falta.

Luego amplía el rol **`jin-core-reader`** con las tres claves y **verifícalo en la base**, porque la UI de Infisical v0.99 falla en silencio (ver `oci-deploy-prep.md`):

```bash
kubectl -n jin exec postgres-0 -- psql -U jin -d infisical -At \
  -c "select permissions::text from project_roles where slug = 'jin-core-reader'"
```

Esperado: `APNS_KEY_ID`, `APNS_TEAM_ID` y `APNS_PRIVATE_KEY` dentro de `secretName $in`.

## 4. Reiniciar jin-core (en la VM)

```bash
KUBECONFIG=$HOME/.kube/config kubectl -n jin rollout restart deploy/jin-core
KUBECONFIG=$HOME/.kube/config kubectl -n jin logs deploy/jin-core | grep -i push
```

Esperado: que **no** aparezca `Push apagado: faltan las claves APNs`.

## 5. App con push (Mac + iPhone)

1. En `Jin_iOS/Jin.xcodeproj`, target **Jin** → *Build Settings* → busca **`JIN_PUSH`** → cámbialo a **`YES`** (Debug y Release).
2. *Signing & Capabilities*: Team = el equipo pagado (no el Personal Team).
3. Conecta el iPhone y dale ▶︎ Run. También puedes pedirle a Claude "reinstala Jin con push".
4. Al abrir, Jin pide permiso para notificaciones: **Permitir**.

## 6. Verificar

- En la app, *Más → Ajustes → Notificaciones* dice **"Activas"**.
- Aviso de prueba:
  ```bash
  curl -s -X POST https://jin.jeanfranck.com/api/push/test -H "Authorization: Bearer <tu JWT>"
  ```
  Esperado: `{"enabled":true,"sent":1,"failed":0}` y la notificación "Jin · prueba" en el iPhone.
- **Estado:** `GET /api/push/status` responde `{"enabled":true}`.

## Si algo falla

| Síntoma | Causa probable |
|---|---|
| `enabled:false` | Faltan las claves en Infisical o en el rol `jin-core-reader`, o no se reinició jin-core. |
| `failed:1`, log `APNs rechazó un aviso (403 InvalidProviderToken)` | Key ID o Team ID equivocados, o la clave está mal pegada. |
| `(400 BadDeviceToken)` y el iPhone desaparece de la lista | Token de sandbox contra producción o al revés. Reinstala desde Xcode (Debug = sandbox). |
| `(400 DeviceTokenNotForTopic)` | El bundle id no es `com.jeanfranck.jin`. Si lo cambiaste, carga `APNS_BUNDLE_ID` con el nuevo. |
| Xcode: "Personal development teams do not support Push Notifications" | Sigue seleccionado el Personal Team (paso 5.2). |

Para volver a apagarlo: `JIN_PUSH = NO` en la app, o borrar las tres claves de Infisical y reiniciar jin-core. Cualquiera de las dos detiene los avisos.
