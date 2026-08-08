# Runbook: preparar la VM de OCI para el deploy real de Jin

## Para quién es esto

Este documento está escrito para que lo siga **un agente (Claude Code u otro) corriendo con acceso sudo dentro de la VM de producción de Jin**, sin contexto previo de ningún otro documento del proyecto. Si sos ese agente: leé completo antes de ejecutar nada, y **detenete y preguntale al owner** en cualquier punto marcado como `⚠️ GATE`.

**Alcance de este documento:** dejar la VM lista, endurecida, y el clúster K3s corriendo con `jin-core` y `jin-executor` en estado `Ready`. **NO cubre**: el consentimiento OAuth de Google, el alta del webhook de Telegram, ni el smoke test end-to-end — eso es la Fase 7.2 (runbook de activación, `docs/runbooks/activation.md`, todavía no escrito), un paso posterior y separado.

**Fuente de verdad del diseño:** `Jin_Docs/docs/BLUEPRINT.md` (arquitectura completa) y el repo `Jin_Infra` (manifests + scripts reales). Si algo en este documento contradice el código de `Jin_Infra`, **el código manda** — este runbook puede quedar desactualizado, el repo no.

---

## 0. Reglas antes de tocar nada

1. **Nunca corras un comando con `sudo` o que toque red/firewall/disco sin leerlo primero.** Este es un servidor de producción real, no un sandbox descartable.
2. **Nunca imprimas, logees, ni pegues valores de secretos** en ningún output que vaya a persistirse (commits, issues, este mismo tipo de transcript si se guarda).
3. Si un paso requiere un valor que no tenés (una API key, un token), **parate y pedíselo al owner** — no inventes valores placeholder para "seguir avanzando", excepto donde este documento diga explícitamente que un placeholder está bien.
4. Si un comando falla, **no reintentes a ciegas ni saltes el paso** — diagnosticá el error real antes de continuar. Casi todos los scripts de `Jin_Infra/scripts/bootstrap/` son idempotentes (se pueden re-correr sin romper nada), pero eso no es excusa para ignorar un fallo real.
5. **Prioridad de este runbook: dejar la superficie de ataque lo más chica posible.** Ante la duda entre "más cómodo" y "más cerrado", elegí más cerrado — es un servidor de un solo usuario, no hace falta nada abierto que no se use.

---

## 1. Verificar que la VM cumple el spec

Debe ser: OCI *Always Free*, instancia ARM Ampere, **4 vCPU / 24 GB RAM / 200 GB block storage**, Ubuntu 24.04 LTS (`BLUEPRINT.md` §3.1).

```bash
cat /etc/os-release | grep -E "PRETTY_NAME|VERSION_ID"
uname -m          # esperado: aarch64 (ARM)
nproc              # esperado: 4
free -h            # esperado: ~24Gi total
df -h /            # esperado: ~200G total
```

Si algo no coincide, `⚠️ GATE`: confirmá con el owner antes de seguir — el presupuesto de recursos de todo el clúster (`BLUEPRINT.md` §3.1.1) asume estos números exactos.

---

## 2. Diagnóstico: ¿la VM corrió Docker antes?

Docker y K3s pueden convivir, pero Docker deja reglas de `iptables` y (en algunas versiones) una configuración de `cgroup` que puede interferir con el networking de K3s (Flannel/kube-proxy). Si esta VM tuvo Docker instalado para otra cosa, lo más seguro es desinstalarlo por completo antes de instalar K3s.

```bash
which docker 2>/dev/null && docker --version || echo "docker no encontrado en PATH"
systemctl is-active docker 2>/dev/null || echo "servicio docker inactivo/no existe"
sudo docker ps -a 2>/dev/null
sudo ss -tlnp | grep -E ':80 |:443 |:6443 |:10250 ' || echo "puertos de K3s libres"
```

Si Docker está instalado y no hay una razón documentada para mantenerlo corriendo en esta VM (no la hay en el diseño actual — Jin no usa Docker en producción, solo K3s/containerd):

```bash
sudo systemctl stop docker docker.socket containerd 2>/dev/null
sudo apt-get purge -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin 2>/dev/null
sudo apt-get autoremove -y --purge
sudo rm -rf /var/lib/docker /etc/docker
sudo iptables -F && sudo iptables -t nat -F && sudo iptables -t mangle -F && sudo iptables -X
```

`⚠️ GATE`: si hay contenedores corriendo con datos que no están respaldados en otro lado, confirmá con el owner antes de purgar — esto es destructivo.

---

## 3. Hardening del sistema operativo (antes de instalar nada más)

El diseño de Jin asume como regla dura que **"la IP de OCI nunca aparece pública"** (`BLUEPRINT.md` §5.1) — todo el tráfico entra vía Cloudflare Tunnel, que es una conexión **saliente** desde `cloudflared` hacia el borde de Cloudflare, sin necesidad de ningún puerto entrante abierto. Pero esa regla no se cumple sola: si el firewall de la nube deja 80/443/6443 abiertos al público, cualquiera puede pegarle directo a la IP de OCI y saltarse Cloudflare por completo, sin que Traefik ni K3s hagan nada raro — el problema no está en la app, está en qué puede llegar a tocarla. Esta sección cierra ese hueco antes de que exista nada corriendo que proteger.

### 3.1 Firewall de la nube (OCI Security List / Network Security Group) — el control que más importa

Esto se configura en la consola de OCI (Networking → Virtual Cloud Networks → tu VCN → Security Lists o NSGs), **no es un script de esta VM**. Regla objetivo: **deny-all de entrada, salvo SSH (puerto 22)**. Nada de 80, 443, 6443, 10250 ni ningún otro puerto expuesto al `0.0.0.0/0` — Cloudflare Tunnel no lo necesita.

`⚠️ GATE`: si no tenés acceso para revisar/editar esto vos mismo (agente), pedíselo al owner — es un cambio a nivel de cuenta de nube, no algo que se resuelva por SSH.

Verificación real, **desde afuera de la VM** (no desde adentro — un firewall mal configurado se ve "bien" desde dentro):
```bash
# Corré esto desde tu laptop, no desde la VM:
nmap -Pn -p 22,80,443,6443,10250 <ip-publica-de-la-vm>
# Esperado: solo 22 aparece open/filtered de forma intencional. El resto: filtered o closed.
```

### 3.2 Firewall del sistema operativo (`ufw`) — defensa en profundidad

Redundante con 3.1 si ese quedó bien, pero un servidor de un solo usuario no pierde nada por tener las dos capas. Con clúster single-node (este caso), `ufw` no debería interferir con el networking interno de K3s (todo el tráfico pod-a-pod queda en interfaces internas del nodo, no en la NIC pública que `ufw` filtra) — aun así, verificá el clúster después de activarlo, no antes de tenerlo instalado.

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 22/tcp
sudo ufw enable
sudo ufw status verbose
```

**Corré esto recién después de instalar K3s (§7.1)**, y confirmá que `kubectl get nodes` y `kubectl get pods -A` siguen respondiendo bien tras activarlo — si algo se rompe, es más fácil diagnosticarlo con el clúster ya arriba que a ciegas.

### 3.3 SSH — cerrar lo obvio

```bash
sudo grep -E "^PasswordAuthentication|^PermitRootLogin|^PubkeyAuthentication" /etc/ssh/sshd_config
```
Si `PasswordAuthentication` no dice explícitamente `no`, o `PermitRootLogin` no dice `no` (o `prohibit-password`):
```bash
sudo sed -i 's/^#\?PasswordAuthentication.*/PasswordAuthentication no/' /etc/ssh/sshd_config
sudo sed -i 's/^#\?PermitRootLogin.*/PermitRootLogin no/' /etc/ssh/sshd_config
sudo systemctl reload sshd
```
`⚠️ GATE`: antes de recargar `sshd`, confirmá que tenés acceso por llave funcionando (ya lo tenés, es como te conectaste) — si algo sale mal acá y perdés la sesión, podrías quedarte afuera de la VM.

Opcional pero recomendado, dado que la VM va a estar expuesta 24/7 en una IP pública conocida (aunque los puertos de la app estén cerrados, el 22 sigue ahí y va a recibir intentos de bots automatizados constantemente):
```bash
sudo apt-get install -y fail2ban
sudo systemctl enable --now fail2ban
```

### 3.4 Actualizaciones de seguridad automáticas

El sistema va a correr sin supervisión activa la mayor parte del tiempo (criterio de éxito de la Fase 7, `BLUEPRINT.md` §14: "corre autónomamente por 7 días"). Que los parches de seguridad del SO no dependan de que alguien se acuerde de correr `apt upgrade`:
```bash
sudo apt-get install -y unattended-upgrades
sudo dpkg-reconfigure -plow unattended-upgrades
```

### 3.5 Nota sobre el acceso de este mismo agente

Le estás dando a este agente sudo real sobre el servidor para este bootstrap. Una vez que termine (después de §8, verificación final), vale la pena que el owner decida si la llave/sesión usada para instalar sigue viva indefinidamente o si conviene rotarla — el acceso permanente de un agente con sudo no es el mismo nivel de riesgo que el acceso puntual para instalar. Esto es una recomendación, no un `⚠️ GATE` — la decisión es del owner.

---

## 4. Clonar `Jin_Infra`

Es el único repo que hace falta clonar en el servidor — los scripts de bootstrap viven ahí. `Jin_Core`/`Jin_Executor` **no se clonan ni se compilan en el servidor**: sus imágenes Docker ya publicadas en GHCR son lo que el clúster va a correr.

```bash
git clone https://github.com/JFrnck/Jin_Infra.git
cd Jin_Infra
```

---

## 5. `⚠️ GATE` — Confirmar que los PRs necesarios están mergeados

El desarrollo de Jin usa PRs por repo, revisados por el owner antes de mergear a `main`. Los scripts de este runbook asumen que **al momento de ejecutarlos, `main` ya tiene mergeado**:

**Imprescindibles (sin esto, el bootstrap no puede terminar):**
- `Jin_Infra` — PR que agrega los manifests de `jin-core`/`jin-executor` y extiende `02-seed-secrets.sh`/`04-apply-manifests.sh`.
- `Jin_Core` — PR de los health probes (`/health/ready`, `/health/live` — sin esto el Deployment de core nunca pasa el `readinessProbe`) y el PR del Dockerfile multi-stage + job de CI que publica a GHCR (sin esto no existe ninguna imagen que descargar).
- `Jin_Executor` — PR del Dockerfile multi-stage + job de CI que publica a GHCR.

**Recomendado mergear junto con lo anterior** (no bloquean el bootstrap en sí, pero dejan `main` en un estado consistente en vez de parcial): cualquier otro PR abierto en `Jin_Infra`/`Jin_Core`/`Jin_Executor`.

**No hace falta para este runbook** (son independientes del clúster K3s): PRs de `Jin_Web` (se despliega a Cloudflare Pages, fuera del clúster) y de `Jin_CLI` (herramienta cliente).

Verificá el estado real, no asumas:

```bash
gh pr list --repo JFrnck/Jin_Infra --state open
gh pr list --repo JFrnck/Jin_Core --state open
gh pr list --repo JFrnck/Jin_Executor --state open
```

Si hay PRs abiertos de la lista "imprescindibles" de arriba, **parate acá** y avisale al owner — no hay forma de continuar de manera significativa sin esto (`02-seed-secrets.sh` fallaría al no encontrar los Secrets nuevos, `04-apply-manifests.sh` no tendría manifests de `jin-core`/`executor` que aplicar, y aunque los aplicara, no habría ninguna imagen en GHCR que descargar).

---

## 6. Prerrequisitos externos (fuera de cualquier script — pasos manuales del owner)

Estos **no los puede resolver un agente automatizado** porque requieren sesión interactiva en dashboards de terceros o son secretos que solo el owner tiene. Confirmá con el owner que cada uno ya existe antes de seguir:

- [ ] Cuenta de Cloudflare con `jeanfranck.com` y `jinserver.com` gestionados ahí (DNS delegado a Cloudflare).
- [ ] Cloudflare Tunnel creado (Zero Trust → Tunnels) — token del túnel a mano.
- [ ] Cloudflare API Token con scope `Zone.DNS Edit` en ambas zonas (lo usa cert-manager para el challenge DNS-01 de los certificados wildcard).
- [ ] CNAME wildcard `*.jeanfranck.com` y `*.jinserver.com` apuntando al túnel (`<tunnel-id>.cfargotunnel.com`).
- [ ] GitHub PAT con scope `read:packages` (para que el clúster pueda hacer `docker pull` de las imágenes privadas en GHCR).
- [ ] GitHub PAT con scope `repo` (lo usa `05-flux-bootstrap.sh` para el GitOps sobre `Jin_Infra`).
- [ ] Flux CLI instalado a mano en la VM — **es un gate deliberado, no automatizable**: instalar un controlador GitOps con permisos cluster-wide requiere intervención humana consciente. El comando exacto (con verificación de checksum) te lo imprime `scripts/bootstrap/05-flux-bootstrap.sh` si intentás correrlo sin tenerlo instalado — seguí esas instrucciones al pie de la letra, no lo automatices.

Y los valores para los Secrets que pide `02-seed-secrets.sh` (ver el bloque de comentarios al inicio de ese archivo para la lista completa y actualizada — no la dupliques de memoria acá, ese archivo es la fuente de verdad). Cubre desde credenciales de Postgres/Redis hasta las API keys de los proveedores de LLM, Telegram, Google OAuth y Modal.

**Nota importante:** para que los pods de `jin-core`/`jin-executor` lleguen a estado `Ready`, alcanza con que estas variables sean strings no vacíos sintácticamente válidos (la validación al arrancar es de forma, no verifica que la key realmente funcione contra el proveedor real). Los valores **reales y funcionales** para que las integraciones (Google OAuth, Telegram, etc.) sirvan de verdad son necesarios recién en la Fase 7.2. Aun así, si el owner ya tiene los valores reales a mano, usalos desde ahora — evita rotar credenciales dos veces.

---

## 7. Orden de ejecución de los scripts de bootstrap

Todos viven en `Jin_Infra/scripts/bootstrap/`. Corré cada uno completo, revisá su output, y **no sigas al siguiente si el anterior no terminó limpio**.

### 7.1 `01-install-k3s.sh`
Instala K3s en la VM (con `servicelb` deshabilitado — la única entrada al clúster es el Cloudflare Tunnel, no queremos puertos públicos escuchando en la IP de OCI).
```bash
sudo -E bash scripts/bootstrap/01-install-k3s.sh
sudo k3s kubectl get nodes   # esperado: 1 nodo, estado Ready
```
Después de este paso, configurá `kubectl` para el resto de la sesión:
```bash
mkdir -p ~/.kube
sudo k3s kubectl config view --raw > ~/.kube/config
chmod 600 ~/.kube/config
kubectl get nodes
```
**Este es el momento de volver a §3.2 y activar `ufw`** si todavía no lo hiciste — con el clúster ya arriba podés verificar que nada se rompió.

### 7.2 `02-seed-secrets.sh`
Siembra todos los Secrets que necesitan existir antes de que el resto del clúster (incluido Infisical) esté operativo. Exportá **todas** las variables que pide (ver §6) antes de correrlo — el script mismo valida que no falte ninguna y aborta si algo falta, sin aplicar nada a medias.
```bash
export POSTGRES_PASSWORD=... REDIS_PASSWORD=... # etc — todas las de la cabecera del script
bash scripts/bootstrap/02-seed-secrets.sh
```

### 7.3 `03-verify-tunnel-dns.sh`
No crea nada — verifica que el túnel de Cloudflare y el DNS wildcard (configurados a mano en §6) estén realmente funcionando.
```bash
bash scripts/bootstrap/03-verify-tunnel-dns.sh
```
Si falla, el problema está en la configuración del dashboard de Cloudflare, no en la VM — volvé a §6.

### 7.4 Actualizar los tags de imagen placeholder

Antes de aplicar los manifests: `k8s/base/jin-core/deployment.yaml` y `k8s/base/executor/deployment.yaml` tienen la imagen fijada a `PLACEHOLDER_UPDATE_BEFORE_DEPLOY` (buscá ese string exacto). Reemplazalo por el tag real — el SHA del commit de `main` que publicó cada CI:

```bash
# Encontrar el SHA real publicado (mismo SHA que el HEAD de main en cada repo tras el merge)
git ls-remote https://github.com/JFrnck/Jin_Core.git HEAD
git ls-remote https://github.com/JFrnck/Jin_Executor.git HEAD

# Confirmar que la imagen con ese tag existe de verdad en GHCR antes de referenciarla
gh api /orgs/JFrnck/packages/container/jin-core/versions 2>/dev/null | grep -A2 "<sha-de-arriba>" \
  || gh api /users/JFrnck/packages/container/jin-core/versions 2>/dev/null | grep -A2 "<sha-de-arriba>"
```

Editá los dos `deployment.yaml`, commiteá y **pusheá a una rama, no directo a `main`** (mismo flujo de PR que el resto del proyecto — dejale el merge al owner, aunque sea un cambio de una línea).

### 7.5 `04-apply-manifests.sh`
Instala cert-manager y aplica el overlay completo de producción (incluye ahora `jin-core` y `executor`).
```bash
bash scripts/bootstrap/04-apply-manifests.sh
```
Este script ya espera (`rollout status`) a que todos los Deployments/StatefulSets queden listos, incluidos `jin-core` y `executor` — si se cuelga en alguno, el problema está ahí, no sigas.

### 7.6 `05-flux-bootstrap.sh`
Activa GitOps: de acá en adelante, un `git push` a `main` de `Jin_Infra` reconcilia el clúster solo.
```bash
export GITHUB_TOKEN=...   # el PAT con scope repo de §6
bash scripts/bootstrap/05-flux-bootstrap.sh
```
Si `flux` no está instalado, el script se detiene e imprime instrucciones con verificación de checksum — es a propósito (ver §6), no lo saltees ni lo automatices.

### 7.7 `06-prepull-deno.sh` y `07-sysctl-inotify.sh`
Tuning del nodo — pre-descarga la imagen de Deno que usan los pods efímeros de ejecución, y sube el límite de `inotify` (sin esto, el hot-reload de los pods de servicio falla en silencio).
```bash
bash scripts/bootstrap/06-prepull-deno.sh
sudo bash scripts/bootstrap/07-sysctl-inotify.sh
```

---

## 8. Verificación final

```bash
kubectl get pods -A
kubectl -n jin get pods -l app.kubernetes.io/name=jin-core
kubectl -n jin-executor get pods -l app.kubernetes.io/name=executor
kubectl -n jin rollout status deployment/jin-core
kubectl -n jin-executor rollout status deployment/executor

# Health checks reales, desde dentro del clúster:
kubectl -n jin run curl-test --rm -it --image=curlimages/curl --restart=Never -- \
  curl -sf http://jin-core:3000/health/live
kubectl -n jin run curl-test --rm -it --image=curlimages/curl --restart=Never -- \
  curl -sf http://jin-core:3000/health/ready
```

Además de la salud del clúster, volvé a correr la verificación de firewall de §3.1 **desde afuera** ahora que todo está desplegado — es la prueba real de que nada quedó expuesto sin querer:
```bash
nmap -Pn -p 22,80,443,6443,10250 <ip-publica-de-la-vm>
```

Si ambos health checks devuelven `200` y el `nmap` solo muestra el 22, el clúster está desplegado y sano. **Esto NO significa que Jin esté funcional de punta a punta** — faltan los pasos de la Fase 7.2 (OAuth de Google, webhook de Telegram, smoke test) antes de que el sistema pueda operar de verdad.

---

## 9. Qué hacer si algo sale mal

- No hay `readinessProbe` en verde tras varios minutos: `kubectl -n <ns> describe pod <pod>` y `kubectl -n <ns> logs <pod>` — casi siempre es un Secret con un valor mal formado o faltante.
- `ImagePullBackOff`: el `ghcr-pull-secret` está mal (usuario/PAT incorrectos) o el tag de imagen no existe todavía en GHCR (volvé a §7.4).
- Certificados TLS no aparecen: `kubectl get certificate -A` y `kubectl describe certificate <nombre> -n <ns>` — el challenge DNS-01 puede tardar minutos; si falla, revisá el Cloudflare API Token de §6.
- `kubectl` dejó de responder después de activar `ufw` (§3.2): `sudo ufw disable`, confirmá que vuelve, y revisá qué regla faltó antes de reactivarlo.
- Cualquier duda sobre si un paso es seguro de reintentar: es seguro si el script es de `scripts/bootstrap/` (todos son idempotentes por diseño) — no lo es si implica borrar/purgar algo (Docker en §2, cualquier `kubectl delete`).

**Ante cualquier situación no cubierta acá, `⚠️ GATE` — parate y consultá con el owner antes de improvisar.**
