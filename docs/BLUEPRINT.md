# Jin — Blueprint maestro

> **Sistema operativo personal + orquestador de agentes.**
> Único usuario (owner), self-hosted en Oracle Cloud Free Tier, escalado externo a Modal cuando la VM se queda corta.
> Este documento es la fuente de verdad. Cuando cualquier decisión de código entre en conflicto con él, gana el blueprint. Si el blueprint está mal, se actualiza primero — nunca se ignora.
> Ver `ANALISIS.md` para el razonamiento del refinamiento 2026-07 (multi-repo, dominios, sqlite-vec, pods bajo demanda).

---

## 1. Objetivos y no-objetivos

### 1.1 Objetivos

- Automatizar el flujo académico (Canvas de U of Alberta) y de productividad (Gmail, Calendar, Notion) sin intervención manual salvo para decisiones críticas.
- Orquestar agentes efímeros que ejecuten código LLM-generado en aislamiento seguro.
- Mantener control humano explícito sobre toda acción de impacto irreversible.
- Vivir dentro de tiers gratuitos siempre que sea razonable, con capacidad de escalar bajo demanda.

### 1.2 No-objetivos

- No es un producto SaaS. No hay multi-tenancy, no hay onboarding, no hay UI pública.
- No compite con productos como Zapier, n8n o Claude Projects. Es una plataforma personal.
- No busca ejecutar código no-confiable a gran escala. El sandbox es para código _propio_ generado por el LLM, no para código de terceros.
- No es tolerante a fallos avanzada (no HA, no multi-región). Es un sistema personal con backups sólidos.
- No optimiza latencias que el humano no percibe: en un flujo HITL la latencia dominante es la aprobación humana, no el arranque de un pod.

### 1.3 Contexto del usuario

- Un solo usuario (el owner). Estudiante U of Alberta, mudanza a Canadá en próximos meses.
- Uso típico: ~10-50 tareas ligeras al día, ~5-20 tareas pesadas al mes.
- Riesgo aceptable: ~5 minutos de downtime al día, ninguna pérdida de datos.

---

## 2. Vista de arquitectura

Cuatro capas, comunicándose siempre en la dirección declarada:

```
┌──────────────────────────────────────────────────────────────┐
│  INTERFACES         Web · Telegram · CLI                      │
└────────────────────────────┬─────────────────────────────────┘
                             ▼ REST / WebSocket
┌──────────────────────────────────────────────────────────────┐
│  ORQUESTACIÓN (OCI VM)                                        │
│  jin-core (NestJS) · BullMQ+Redis                             │
│  Postgres+pgvector · sqlite-vec (memoria) · Infisical         │
└────────────┬──────────────────────────────┬──────────────────┘
             ▼ HTTP interno                 ▼ SDK
┌───────────────────────────┐   ┌──────────────────────────────┐
│  EJECUCIÓN LOCAL          │   │  ESCALADO EXTERNO            │
│  jin-executor +           │   │  Modal Sandbox (gVisor)      │
│  pods Deno bajo demanda   │   │                              │
└───────────────────────────┘   └──────────────────────────────┘
             ▼                                 ▼
┌──────────────────────────────────────────────────────────────┐
│  PERSISTENCIA           Cloudflare R2 · Cloudflare Tunnel     │
└──────────────────────────────────────────────────────────────┘
```

**Regla arquitectónica dura:** jin-core _nunca_ habla con el socket de Docker/K3s directamente. Solo el Executor (microservicio separado con ServiceAccount propia y RBAC restrictivo) puede crear/destruir pods. Esta separación no es negociable — es la única defensa arquitectónica contra que un prompt injection en core escale a "borrar todo el clúster".

**Estructura de código: multi-repo.** Cada app vive en su propio repositorio — `Jin_Core`, `Jin_Executor`, `Jin_Web`, `Jin_CLI`, `Jin_Infra`, `Jin_Docs`, clonados juntos en la carpeta local `Jin/` — con su CI, sus dependencias y su versión de Node (`.nvmrc`). Los contratos entre repos son OpenAPI generado desde código, no paquetes compartidos. **Convención de nombres:** repos/carpetas en `Jin_X`; todo nombre runtime de K8s (namespaces, services, imágenes, bucket) en minúscula kebab (`jin-core`, `jin-executor`) porque K8s lo exige. Ver sección 4.6 y `AGENTS.md` sección 4.

---

## 3. Infraestructura y persistencia

### 3.1 VM principal

- **Oracle Cloud Infrastructure Always Free**, instancia ARM Ampere.
- Recursos: 4 vCPU · 24 GB RAM · 200 GB block storage.
- OS: Ubuntu 24.04 LTS.
- Región: la más cercana a tu ubicación actual con Always Free disponible.

#### 3.1.1 Presupuesto de recursos (obligatorio)

Total disponible tras reservar SO/K3s: **20 GB RAM, 3.5 vCPU**.

| Componente              | Requests (RAM) | Limits (RAM)   | Requests (CPU) | Limits (CPU)    |
| ----------------------- | -------------- | -------------- | -------------- | --------------- |
| Postgres (con pgvector) | 4 Gi           | 6 Gi           | 500m           | 1500m           |
| Redis                   | 512 Mi         | 1 Gi           | 100m           | 500m            |
| Infisical               | 256 Mi         | 512 Mi         | 50m            | 200m            |
| jin-core (1 réplica) | 512 Mi         | 1 Gi           | 200m           | 800m            |
| jin-executor         | 256 Mi         | 512 Mi         | 100m           | 400m            |
| Traefik                 | 128 Mi         | 256 Mi         | 50m            | 200m            |
| cloudflared             | 64 Mi          | 128 Mi         | 50m            | 100m            |
| Prometheus              | 512 Mi         | 1 Gi           | 100m           | 300m            |
| Loki                    | 256 Mi         | 512 Mi         | 100m           | 300m            |
| Grafana                 | 256 Mi         | 512 Mi         | 50m            | 200m            |
| cert-manager            | 64 Mi          | 128 Mi         | 50m            | 100m            |
| Flux                    | 128 Mi         | 256 Mi         | 50m            | 200m            |
| **agents-sandbox pool** | —              | **6 Gi techo** | —              | **1500m techo** |

Los pods efímeros consumen del pool `agents-sandbox` mediante ResourceQuota a nivel de namespace, no requests individuales. **La quota es un techo de ráfaga, no una reserva: sin tareas corriendo, el consumo del namespace es cero** (no hay warm pool — ver 4.4).

jin-core corre con **1 réplica**: para un solo usuario, una segunda réplica siempre encendida es redundancia sin retorno. El rolling update sigue siendo sin downtime vía `maxSurge: 1, maxUnavailable: 0` (ver 12.2).

Ver `AGENTS.md` sección 4.4 para la regla dura de aplicación, y ADR 0002 para el análisis de por qué es obligatorio y no recomendado.

### 3.2 Runtime

- **K3s** (Rancher's lightweight Kubernetes) como orquestador.
- **Traefik** (bundled con K3s) como Ingress controller.
- **cert-manager** con challenge DNS-01 para wildcard certs de Let's Encrypt.
- **cloudflared** como pod para el Cloudflare Tunnel.

### 3.3 Base de datos

- **Postgres 16** con extensión `vector` (pgvector 0.7+).
- Corre como StatefulSet en K3s con PersistentVolume de 50 GB.
- Índice HNSW sobre embeddings; IVFFlat como fallback si RAM aprieta.
- Connection pooler: PgBouncer en modo `transaction`.

**Por qué pgvector y no Qdrant/Supabase:**

- Datos relacionales (tareas, usuarios, audit log) y vectoriales conviven en la misma base con transacciones ACID. Un `JOIN` entre `tasks` y `task_embeddings` es SQL nativo, no una sincronización entre dos servicios.
- A escala <5M vectores, la diferencia de latencia contra Qdrant es imperceptible.
- Sin dependencia de un vendor (Supabase pausa proyectos free tras 7 días, y 500 MB no alcanzan para el volumen de embeddings de correos + PDFs de Canvas).
- Migrar a Supabase o Qdrant en el futuro es `pg_dump` — cero lock-in.

#### 3.3.1 Memoria extendida local: sqlite-vec

Complementa a pgvector, no lo reemplaza. Frontera:

- **pgvector = corpus.** Correos indexados, PDFs de Canvas, notas. Grande, con JOINs relacionales.
- **sqlite-vec = memoria del agente.** Hechos sobre el owner, preferencias, resúmenes episódicos de sesiones, decisiones y lecciones. Pequeña (≤100k vectores), curada, caliente: se lee al inicio de cada sesión de agente (recall) y se escribe al cierre (consolidación).

Diseño:

- Archivo único `memory.db` en el PersistentVolume de jin-core. Acceso vía `better-sqlite3` + extensión `sqlite-vec`, WAL mode, un solo writer.
- **Solo jin-core** (módulo `src/memory/`) lee/escribe. Los pods efímeros jamás lo tocan.
- Búsqueda: KNN brute-force + filtros por columnas de metadata (estable en sqlite-vec; sus índices ANN siguen en alpha — no depender de ellos).
- Metadata mínima por entrada: `tipo`, `fuente`, `fecha`, `modelo_embedding` (permite migrar de modelo de embeddings sin adivinar).
- Backup: incluido en el cron nocturno (ver 3.5). Restaurable con un `cp`.
- Si algún día supera la escala razonable, migrar a pgvector es un script.

### 3.4 Cache y colas

- **Redis 7** como StatefulSet con AOF persistence habilitada.
- **BullMQ** sobre Redis para todas las colas de tareas asíncronas.

### 3.5 Backups

- Cron diario a las 03:00 local:
  - `pg_dump --format=custom` de Postgres.
  - Snapshot del volumen Redis (`SAVE` + copia del RDB).
  - `sqlite3 memory.db ".backup"` de la memoria extendida.
  - Snapshot del state de K3s (etcd).
- Cifrado con `age` usando llave master guardada en Infisical.
- Subida a Cloudflare R2 (bucket `jin-backups`).
- **Retención:** 7 diarios · 4 semanales · 3 mensuales.
- **Prueba de restore mensual automatizada:** cron el día 1 de cada mes levanta un contenedor Postgres temporal, aplica el dump más reciente, corre `SELECT COUNT(*)` en tablas críticas, valida, y notifica por Telegram el resultado. **Un backup nunca probado no es un backup.**

### 3.6 Almacenamiento externo

- **Cloudflare R2** (10 GB free · 1M writes · 10M reads · egress $0).
- Usos: dumps cifrados, artefactos de agentes, exports, assets pesados del dashboard.
- Cuando la DB dumped supere 7 GB, migrar backups viejos (>30 días) a Backblaze B2 como cold storage.

---

## 4. Orquestación y ejecución aislada

### 4.1 jin-core (orquestador principal)

- Node.js 24 LTS. Repo propio: `Jin_Core`.
- Corre como Deployment con 1 réplica (rolling update con surge, ver 12.2).
- Responsabilidades:
  - Auth (single user, JWT firmado con llave en Infisical).
  - API REST + WebSocket para las tres interfaces. Publica su contrato OpenAPI (ver 4.6).
  - Cron jobs vía `@nestjs/schedule` (Shadowing Académico, backups, health checks).
  - Encolado de tareas a BullMQ.
  - Router de modelos LLM.
  - Registry de herramientas (tools) y clasificador HITL.
  - Memoria extendida (sqlite-vec, módulo `memory`).
  - Bot de Telegram como módulo interno.
  - Comunicación con Executor vía HTTP interno.
- **Prohibiciones:**
  - No importa `dockerode` ni `@kubernetes/client-node`.
  - No tiene acceso al socket de Docker/K3s.
  - No corre con `hostNetwork` ni con SecurityContext privilegiado.

### 4.2 jin-executor (microservicio de ejecución)

- Node.js 24 LTS, servicio separado. Repo propio: `Jin_Executor`.
- Corre como Deployment con 1 réplica en el namespace `jin-executor`.
- ServiceAccount propia con Role restrictivo:
  - `create`, `get`, `list`, `delete` de Pods **solo en el namespace `agents-sandbox`**.
  - Nada más. Sin acceso a Secrets, ConfigMaps, Deployments, ni otros namespaces.
- Endpoint HTTP interno recibe peticiones de core:
  ```
  POST /execute
  { tool: string, code: string, env: object, timeout: number, remote: boolean }
  ```
- Valida contra whitelist de herramientas antes de crear el pod.
- Si `remote: true`, envía a Modal en lugar de crear pod local.
- Publica su propio contrato OpenAPI; core genera el cliente desde ahí (ver 4.6).

### 4.3 NetworkPolicies (aislamiento de red)

- Namespace `jin`: core, Redis, Postgres. Puede recibir de `jin-frontend`. No puede iniciar tráfico a `agents-sandbox`.
- Namespace `jin-executor`: Executor. Recibe solo de `jin`. Puede crear pods en `agents-sandbox`.
- Namespace `agents-sandbox`: pods efímeros Deno. Solo pueden salir por egresos específicos declarados (URLs whitelisted). No pueden ver otros namespaces.

### 4.4 Tier local: pods Deno bajo demanda

- Deno 2.x, elegido por seguridad por defecto (flags `--allow-net` explícitos).
- **Sin warm pool.** El Executor crea el pod cuando llega la tarea, ejecuta, entrega el resultado y lo destruye. Racional: con un solo usuario y flujos gateados por HITL, la latencia dominante es la aprobación humana; mantener pods idle era gastar RAM en ahorrar segundos que nadie percibe.
- Imagen Deno pre-descargada en el nodo (pre-pull en el bootstrap y tras cada upgrade de imagen) → latencia de arranque ~2-5 s.
- Timeout máximo: 5 minutos. Tareas más largas rutean a Modal.

### 4.5 Tier de escalado: Modal

- Modal Starter tier: **$30/mes de créditos gratuitos recurrentes**, gVisor sandbox, sub-second cold start.
- Cuándo escala a Modal (decisión automática por el Executor):
  - Tarea estimada >5 min.
  - Requiere >4 GB RAM.
  - Requiere librerías Python científicas (pandas + numpy + torch).
  - Requiere GPU (raro pero posible).
- Modal SDK invocado desde el Executor. El código a ejecutar se transmite en la petición.
- Los créditos $30 cubren aproximadamente 100 CPU-horas mensuales — más que suficiente para uso personal esperado.

### 4.6 Contratos entre repos (reemplazo del monorepo)

- **No hay paquetes compartidos.** Los packages `shared-types`, `shared-config` y `shared-audit` del diseño original se disolvieron (ver `ANALISIS.md` sección 3).
- Jin_Core y Jin_Executor generan `contracts/openapi.json` desde su código (`@nestjs/swagger`) en CI.
- Consumidores generan tipos con `openapi-typescript` (`pnpm generate:api`): web y cli desde el contrato de core; core desde el contrato del executor.
- **Prohibido copiar tipos a mano entre repos.** Si el contrato cambia, el consumidor regenera y su `tsc` detecta el breaking change.
- Cambios de API entre repos: retrocompatibles, o en dos pasos (expand → contract).

---

## 5. Dominios, exposición y TLS

### 5.1 Cloudflare Tunnel (default, no opcional)

- Daemon `cloudflared` como pod en K3s con token guardado en Infisical.
- Expone Traefik a Cloudflare por túnel; la IP de OCI nunca aparece pública.
- Beneficios extra: DDoS gratis, Access rules como capa adicional (ej. token en móvil para acceder al dashboard).

### 5.2 Los dominios y sus roles

Dos dominios registrables, tres zonas de confianza:

| Zona | Host | Qué sirve | Confianza |
| --- | --- | --- | --- |
| Identidad del owner | `jeanfranck.com` (raíz y paths) | Portafolio personal. **No es parte de Jin** — no se despliega nada del sistema aquí. | Del owner |
| Núcleo de Jin | `jin.jeanfranck.com` | Dashboard (Cloudflare Pages) + API bajo el path `/api` (ruteada al túnel) | Código revisado |
| Sandbox | `jinserver.com` | `<slug>.jinserver.com` — previews, applets, share links | **Generado por agentes, sin revisar** |

**La API va como path (`jin.jeanfranck.com/api`), no como subdominio.** Siendo un sistema de un solo usuario, un único origen elimina CORS por completo y permite que la cookie de sesión sea *host-only* (`__Host-` prefix, sin atributo `Domain`) en vez de tener que abrirse a `.jeanfranck.com`. Se implementa con una route rule de Cloudflare: `/api/*` al túnel, el resto a Pages.

**Regla dura de separación de origen:** contenido generado por agentes se sirve **solo** desde `jinserver.com`, jamás desde `jeanfranck.com` ni ninguno de sus subdominios. Comparten túnel pero no dominio registrable, y eso es justamente el punto: el navegador trata el par como *cross-site*, así que la cookie de sesión del dashboard nunca viaja a una petición originada por código que escribió un LLM. Es el mismo patrón que `githubusercontent.com` frente a `github.com`.

El escenario concreto que esto previene: un agente lee un input externo envenenado con una inyección de prompt y genera una app cuyo JS hace `fetch('.../api/tools/sendEmail', {credentials:'include'})`. Con dominios separados el navegador no adjunta la cookie y la petición muere sola, sin depender de que la validación de `Origin` del servidor esté bien escrita. **Falla seguro, no abierto.**

Aunque el dominio sandbox lleve la marca en el nombre, es desechable por diseño: su raíz no sirve nada y las previews no llevan branding de Jin ni enlaces de vuelta al panel, para no transferirle confianza.

### 5.3 DNS y certificados

- Ambos dominios con Cloudflare como DNS.
- CNAME wildcard `*.jeanfranck.com` y `*.jinserver.com` apuntando al túnel.
- **cert-manager** con `ClusterIssuer` de Let's Encrypt usando challenge **DNS-01** (obligatorio para wildcards). Credenciales Cloudflare API en Infisical.

### 5.4 Ingress dinámico

- Cuando un agente levanta una mini-app (ej. React de prueba) en puerto 8081, core crea un `IngressRoute` de Traefik con el subdominio `<slug>.jinserver.com`.
- **El slug lleva sufijo aleatorio** (`residuos-educan-a7f3k9`), no solo el nombre del proyecto: delante de una preview no hay login, así que la única barrera contra quien adivine la URL es la entropía del subdominio. El prefijo legible se conserva para poder distinguirlas de un vistazo.
- Los subdominios de agentes tienen **TTL obligatorio** (definido por el owner al aprobar, ej. 4 h; cap máximo 24 h) y los limpia un reaper por cron. Extender el tiempo requiere una aprobación nueva — no existe camino por el cual un servicio quede corriendo indefinidamente.
- Opcionalmente detrás de Cloudflare Access para las previews que el owner no quiera compartir con nadie.

**Limitación conocida:** dos previews bajo `jinserver.com` siguen siendo *same-site entre sí*, así que una puede hacer cookie tossing sobre otra. Aislarlas de verdad requiere meter `jinserver.com` en la Public Suffix List (lo que hace `github.io`); es gratis pero tarda meses en propagar. Queda como item de hardening de Fase 7.3, no bloquea nada.

---

## 6. Capa de IA

### 6.1 ModelProvider (abstracción anti-obsolescencia)

- Interface uniforme `ModelProvider` con implementaciones para Anthropic y Google.
- **No hay modelos hardcoded en el código.** El registry vive en `config/models.yaml` (montado como ConfigMap) y se puede recargar sin redeploy.
- Cada tarea declara un `TaskProfile` (razonamiento, contexto largo, extracción rápida, etc.) y el router elige el modelo según el profile + budget disponible.

### 6.2 Modelos recomendados (Julio 2026)

Ver `MODEL_ROUTING.md` para la tabla completa. Resumen:

- **Razonamiento y código:** Claude Opus 4.8 (`claude-opus-4-8`) o Sonnet 5 (`claude-sonnet-5`).
- **Contexto masivo (>500k tokens):** Gemini 3.1 Pro (`gemini-3.1-pro`, 2M context).
- **Rápido/barato:** Claude Haiku 4.5 (`claude-haiku-4-5`) o Gemini 3.5 Flash (`gemini-3.5-flash`).

### 6.3 Structured outputs

- Toda respuesta del LLM que el sistema vaya a procesar programáticamente usa **tool-use API** (function calling), no "responde en JSON". El constrained decoding garantiza JSON válido.
- Respuestas dirigidas al humano (chat, reportes) pueden ser markdown o texto libre.

### 6.4 RAG y memoria

- **Corpus propio** (correos indexados, notas, tareas): embeddings con `text-embedding-3-large` de OpenAI o `voyage-3` de Voyage AI, guardados en pgvector.
- **Memoria del agente** (hechos, preferencias, episodios): sqlite-vec local (ver 3.3.1). Recall al inicio de sesión, consolidación al cierre.
- **Documentación técnica externa** (React, Tailwind, Next.js, etc.): **MCP servers oficiales** (Context7, etc.). No mantenemos pipeline propio de scraping.

### 6.5 Defensa contra prompt injection

Ver `AGENTS.md` sección "Seguridad" para las reglas duras. Resumen:

1. Todo input externo se envuelve en `<untrusted_content source="...">...</untrusted_content>`.
2. El system prompt declara explícitamente que las instrucciones dentro de esos tags no son órdenes.
3. Sanitizador pre-indexación filtra patrones sospechosos.
4. Cada card HITL muestra "Inputs externos que influyeron en esta decisión" para trazabilidad.

---

## 7. Integraciones

### 7.1 Canvas LMS

- **Inicial:** solo instancia de prueba (Canvas Free-for-Teachers o sandbox propio).
- **Cuando esté en U of A:** revisar ToS académico antes de activar. Si permite: Personal Access Token → Infisical → integración vía API REST.
- Shadowing Académico: cron 00:00 local. Detecta tareas nuevas, anuncios, cambios en archivos. Genera bloques de estudio en Google Calendar (con HITL `notify`). Alerta matutina 06:00 con resumen de prioridades.

### 7.2 Google Workspace

- **Fase 1 (Perú, actual):** OAuth en modo Testing con la cuenta personal Google del owner. Aceptamos renovación manual de token cada 7 días (job en BullMQ que avisa 24 h antes).
- **Fase 2 (Alberta, futuro):** evaluar cambio a app password + IMAP/SMTP para correo si TI de U of A permite, y mantener OAuth solo para Calendar y Drive.
- Scopes mínimos necesarios; nada extra "por si acaso".

### 7.3 GitHub

- **GitHub App** (no PAT) con permisos scoped a repos específicos.
- Tokens de instalación efímeros (1 hora de vida) generados on-demand por el Executor cuando un pod necesita hacer git ops.
- El token se inyecta como env var al pod, no se persiste.

### 7.4 Notion

- Integration token internal, scoped a un workspace específico.
- Usos: logs de entrenamiento físico, resúmenes académicos, base de enlaces.

### 7.5 Telegram

- Bot creado con BotFather. Token en Infisical.
- Librería: **grammY** (más moderno que telegraf, mejor tipado).

---

## 8. Interfaces

### 8.1 Web Dashboard

- **Stack:** Vite + React 19 + React Router v7 (framework mode). Repo propio: `Jin_Web`.
- **Compilación:** static build servido desde **Cloudflare Pages** en `jin.jeanfranck.com` (fuera de la VM, cero costo, cero consumo de OCI).
- **Editor de código:** Monaco Editor con Zone Widgets para comentarios inline de la IA.
- **Applets:** iframes con `sandbox="allow-scripts"` cargando subdominios `app-{uuid}.jinserver.com` (cross-origin por diseño, ver 5.2).
- **Comunicación con backend:** REST para comandos, WebSocket para eventos en tiempo real (progreso de tareas, aprobaciones pendientes). Tipos generados desde el OpenAPI de core.

#### 8.1.1 PWA (decisión del owner, 2026-08-02)

El dashboard se instala como PWA en el móvil del owner. El motivo no es offline — es que **aprobar desde el móvil hoy solo se puede por Telegram**, y una app en la pantalla de inicio (con push a futuro) es el camino natural para que el dashboard cubra ese caso sin depender del bot.

Tres reglas duras, porque un PWA mal configurado degrada la seguridad del sistema:

1. **El service worker jamás cachea `/api/*`.** Solo el app shell (HTML, JS, CSS, íconos). Un `runtimeCaching` genérico tipo Workbox persistiría aprobaciones pendientes, payloads de correos y el audit log en `CacheStorage` — datos sensibles escritos a disco que **sobreviven al logout**, porque borrar la cookie no borra el cache. La sesión debe poder terminarse de verdad.
2. **Offline es solo lectura degradada. Prohibido encolar mutaciones.** Nada de background sync para aprobar/rechazar: una aprobación emitida offline y sincronizada tres horas después aprueba una acción cuyo estado ya cambió, o cuyo TTL de 24 h (9.4) ya venció. El HITL asume decisión *presente* sobre estado *presente*. La UI bloquea explícitamente aprobar/rechazar sin conexión — es un requisito, no una limitación técnica.
3. **Un service worker por origen, y el origen es el confiable.** El SW vive en `jin.jeanfranck.com` y jamás se registra desde contenido de agentes. Esto ya lo garantiza 5.2 (las previews viven en otro dominio registrable), y es exactamente el ataque que 5.2 describe: un service worker con scope sobre el origen de la sesión intercepta cada petición que se hace desde ahí.

**Notificaciones push: fuera de alcance hasta que exista soporte en `jin-core`.** Requiere VAPID keys, persistir suscripciones y lógica de envío — nada de eso está construido. Cuando se implemente, es el reemplazo natural del canal de aprobación de Telegram, no un duplicado más.

### 8.2 Telegram Bot

- grammY con webhook (no polling) para bajo consumo. Módulo interno de jin-core.
- Comandos: `/tasks`, `/approve`, `/reject`, `/status`, `/budget`, `/audio` (transcribe y guarda en Notion).
- Cards de HITL con botones inline (`Aprobar`, `Rechazar`, `Ver detalles`).

### 8.3 CLI

- **Ink** + **Inquirer.js** para menús navegables con teclas de flecha. Repo propio: `Jin_CLI`.
- Se conecta al mismo API de core (tipos generados desde su OpenAPI).
- Casos de uso: desarrollo local, debugging, admin tasks (rotar tokens, forzar backup, ver logs recientes).

---

## 9. Sistema HITL (Human-in-the-Loop)

Este es el sistema de seguridad más importante. Su diseño está detallado aquí porque un error acá tiene consecuencias irreversibles.

### 9.1 Cuatro niveles de acciones

| Nivel          | Comportamiento                              | Ejemplos                                                                                           |
| -------------- | ------------------------------------------- | -------------------------------------------------------------------------------------------------- |
| `auto`         | Ejecuta sin notificar                       | Leer correos, indexar docs, buscar en Google Calendar, generar embeddings                          |
| `notify`       | Ejecuta y envía notificación post-hoc       | Crear evento en Calendar, añadir tarea a Notion, `git commit` local                                |
| `confirm`      | Espera 1 aprobación antes de ejecutar       | `git push`, responder correo, apagar contenedor, cambio en config                                  |
| `dual-confirm` | Requiere 2 aprobaciones separadas por ≥30 s | Borrar archivos, `git push --force`, drop de tabla, revocar tokens, cambiar HITL level de una tool |

### 9.2 Los 30 segundos del dual-confirm

Son intencionales. Evitan el "apruebo todo por reflejo" cuando estás en el móvil. Después de la primera aprobación, la segunda card aparece exactamente 30 s más tarde, mostrando el mismo plan. Si apruebas ambas rápido sin leer, la responsabilidad es tuya, pero al menos hubo pausa forzada.

### 9.3 Clasificador HITL

- **Ubicación:** módulo `hitl-classifier` en jin-core.
- **Datos:** cada tool en el registry declara su `hitlLevel` como parte de su definición estática.
- **Prohibido:** el LLM no puede cambiar el `hitlLevel` de una tool en runtime.
- **Verificación:** unit tests obligatorios cubriendo 100% de la matriz tool × level. Si el LLM decide llamar una tool `dual-confirm` como si fuera `auto`, el clasificador rechaza.

### 9.4 Timeout de aprobación

- TTL default: 24 horas.
- Al expirar:
  - **Reversible/informativo** (responder correo, comentario en Notion): descarta silenciosamente, notifica.
  - **Con deadline** (submit de tarea de Canvas): escala. Segundo aviso a las 12 h con urgencia. Sin respuesta → marca `abandoned`, guarda estado para revisión manual, notifica error.
  - **Sin auto-aprobación jamás.** El timeout nunca ejecuta la acción.

### 9.5 Audit log inmutable

Tabla `audit_log` (ver ADR 0002 para el porqué de `request_id`):

```sql
CREATE TABLE audit_log (
  id BIGSERIAL PRIMARY KEY,
  request_id UUID NOT NULL,         -- correlaciona todas las filas de un mismo evento
  timestamp TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  actor TEXT NOT NULL,              -- 'system' | 'user' | 'agent:{name}'
  action_type TEXT NOT NULL,        -- 'tool_call' | 'approval' | 'rejection' | 'timeout'
  tool_name TEXT,
  inputs_hash TEXT NOT NULL,        -- sha256 de los inputs
  plan_summary TEXT,
  approval_status TEXT NOT NULL,    -- 'auto' | 'notified' | 'pending' | 'approved' | 'rejected' | 'timeout' | 'abandoned'
  approver TEXT,
  external_inputs_summary TEXT,     -- qué correos/docs influyeron
  prev_hash TEXT NOT NULL,
  current_hash TEXT NOT NULL
);
```

- `current_hash = sha256(prev_hash || row_data_canonical)`.
- **Insert-only, siempre.** Una acción con nivel `confirm`/`dual-confirm` genera una fila `pending` al llegar, y una fila NUEVA (nunca un UPDATE de la anterior) cuando se resuelve (`approved`/`rejected`/`timeout`/`abandoned`). `request_id` (mismo UUID en todas las filas del mismo evento) es lo que permite reconstruir el ciclo de vida completo sin romper el hash chain — actualizar una fila histórica invalidaría su hash y el de todas las siguientes (regla de oro #12 / AGENTS.md 5.6).
- **Estado "pendiente" vive aparte, en la tabla mutable `pending_approvals`** (mismo `request_id` como PK), no en `audit_log`. Se borra o marca resuelta cuando se escribe la fila terminal en el log inmutable. Sobrevive a un restart del pod (persistida, no en memoria) — con 1 réplica y rolling updates, una aprobación en memoria se perdería silenciosamente.
- Cron diario 04:00 valida la cadena completa. Si detecta corrupción → alerta Telegram + escritura bloqueada hasta intervención manual.
- Replicación horaria a R2 cifrado.

### 9.6 Budget guard

Tres niveles de presupuesto, todos configurables en `config/budget.yaml`:

**Per-session** — cada sesión/tarea tiene:

- Presupuesto máximo de tokens (default 500k input + 100k output).
- Al 80% → mensaje del sistema "wrap up".
- Al 100% → detención, requiere continuación explícita.

**Daily** — límite global de tokens y dólares:

- Default: 5M tokens/día, $10 USD/día.
- Al 80% → notificación.
- Al 100% → solo tools `auto` con modelos Haiku/Flash hasta reset (00:00 local).

**Hard cap (kill switch)** — detección de runaway:

- Si en 1 hora se consumen >2× de lo consumido en las 24 h previas → pausa TODOS los agentes, notifica Telegram, requiere unpause manual con comando `/unpause` que a su vez es `confirm`.

### 9.7 Rate limiting

- Por-tool: cada tool declara su `rateLimit` (ej. `sendEmail: 5/hour`).
- Backing store: Redis con sliding window.
- Aplicado tanto por el LLM (no puede llamar más allá del límite) como por el usuario (no puede aprobar en burst).

---

## 10. Observabilidad

### 10.1 Métricas

- **Prometheus** como servidor de métricas (ligero, retención local 15 días).
- **Grafana** para dashboards.
- Métricas clave:
  - `tokens_consumed_total{model, task_type}` — counter
  - `tool_latency_seconds{tool, hitl_level}` — histogram
  - `hitl_approval_rate{level}` — gauge
  - `bullmq_queue_depth{queue}` — gauge
  - `pod_cpu_usage_bytes{namespace}` — gauge
  - `rag_hit_ratio` — gauge

### 10.2 Logs

- **Loki** para agregación (retención 30 días local, backup a R2).
- Formato JSON estructurado en todos los servicios.
- Campos obligatorios: `timestamp`, `service`, `trace_id`, `session_id`, `level`, `message`, `context`.

### 10.3 Tracing

- **OpenTelemetry** SDK en core y Executor.
- Backend: Grafana Tempo (self-hosted en K3s) o Honeycomb Free Tier.
- Traces obligatorios: cada tool call, cada llamada LLM, cada HITL approval.
- Nota: `ANALISIS.md` recomienda posponer Tempo hasta después de la Fase 4; decisión pendiente del owner.

### 10.4 Alertas (via Alertmanager → Telegram)

- Budget al 80% del daily.
- Budget al 100% del daily.
- Runaway detected.
- Backup failed.
- Restore test failed.
- Audit chain corrupted.
- Cert expiration <7 días.
- Pod restarts >3 en 10 min.

---

## 11. Gestión de secretos

- **Infisical** self-hosted como Deployment en K3s (open-source, panel usable, SDK Node).
- Todos los secretos viven en Infisical: API keys, PATs, tokens OAuth, JWT signing key.
- core y Executor los cargan al startup vía SDK; nunca están en env plain.
- **Master key de Infisical** cifrada con `age`, guardada en dos lugares:
  - Llavero local del owner (Bitwarden / 1Password).
  - Backup encriptado en R2 con llave separada.
- **Rotación:** trimestral obligatoria. Job en BullMQ marca tokens como "rotate soon" 7 días antes de expirar.

---

## 12. Deployment y CI/CD

### 12.1 GitOps con Flux

- Repo `Jin_Infra` con manifests de Kubernetes (Kustomize), scripts de bootstrap y de backup.
- Flux corre en K3s, hace polling cada 5 min.
- `git push` a `main` → detecta cambios → aplica cambios → notifica Telegram.

### 12.2 Rolling updates

- Todos los Deployments tienen `maxSurge: 1, maxUnavailable: 0`.
- Con 1 réplica esto sigue dando cero downtime: K8s levanta la réplica nueva y solo entonces mata la vieja.
- ReadinessProbe obligatoria; sin ella no pasa el linter de CI.

### 12.3 Feature flags

- ConfigMap `feature-flags.yaml`. Reload en caliente vía SIGHUP en core.
- Uso: activar/desactivar integraciones, cambiar HITL levels de tools individuales (con dual-confirm), rutear entre modelos.

### 12.4 CI (GitHub Actions, por repo)

Cada repo tiene su propio pipeline — un fallo en un repo jamás bloquea a los demás:

- **Jin_Core / Jin_Executor:** lint, typecheck, unit tests, integration tests (Postgres + Redis en Docker), generación y commit de `contracts/openapi.json`, build de imagen, push a GHCR.
- **Jin_Web:** lint, typecheck, `generate:api` contra el contrato de core (el drift rompe el build aquí, no en producción), tests, deploy a Cloudflare Pages.
- **Jin_CLI:** lint, typecheck, `generate:api`, tests.
- **Jin_Infra:** kube-linter/kubeval sobre los manifests; Flux hace el deploy.
- Node por repo vía `.nvmrc` (`setup-node` lo lee). Workflows comunes como reusable workflows (`workflow_call`) para no duplicar boilerplate.

---

## 13. Testing

### 13.1 Obligatorio

- **HITL classifier:** 100% de cobertura de la matriz tool × level.
- **Audit log hash chain:** tests de mutación (modificar una row rompe el chain).
- **Budget guard:** simulación de consumo creciente disparando cortes en 80%, 100% y runaway.
- **Executor RBAC:** tests que verifican que peticiones fuera de whitelist son rechazadas.
- **Prompt injection golden set:** corpus de ~50 prompts adversariales conocidos; el sistema debe manejarlos correctamente.

### 13.2 Recomendado

- E2E: flujo Canvas → resumen → HITL → aprobación con mocks.
- Load test: 100 tareas concurrentes en BullMQ.
- Chaos test mensual: `kubectl delete pod` random al pod de core; verificar recovery <10 s.

---

## 14. Roadmap (12 semanas)

Ordenado por dependencia y riesgo, no por atractivo.

### Fase 1: Base (semanas 1-2)

- Provisionar OCI VM.
- Instalar K3s + Traefik + cert-manager.
- Configurar Cloudflare Tunnel + wildcard DNS en ambos dominios.
- Deployment de Postgres + Redis + Infisical.
- Setup de backups automáticos a R2.
- Setup de Prometheus + Grafana + Loki mínimos.
- **Criterio de éxito:** dominios responden con TLS, backups suben cifrados, dashboard de Grafana muestra métricas de K3s.

### Fase 2: Núcleo (semanas 3-4)

- Repos Jin_Core y Jin_Executor scaffoldeados (CI propio, `.nvmrc`, OpenAPI export).
- Jin_Core: boilerplate + auth + BullMQ.
- Executor separado con RBAC.
- ModelProvider abstracto + implementaciones Anthropic y Google.
- Telegram bot con grammY.
- HITL básico (`auto` y `confirm`) + audit log con hash chain.
- **Criterio de éxito:** puedes pedir por Telegram "resume estos textos", los procesa con Claude, la respuesta llega, y el audit log tiene la entrada correcta.

### Fase 3: Primera integración end-to-end (semana 5)

- Canvas LMS (instancia de prueba).
- Shadowing Académico completo con HITL.
- **Criterio de éxito:** un ciclo nocturno completo funciona: Canvas → análisis → propuesta → notificación → aprobación → creación en Calendar.

### Fase 4: Segunda integración + guardrails avanzados (semanas 6-7)

- Google Calendar + Gmail.
- Niveles `notify` y `dual-confirm`.
- Budget guard completo con kill switch.
- Rate limiting per-tool.
- Módulo de memoria extendida con sqlite-vec (recall al inicio de sesión, consolidación al cierre).
- **Criterio de éxito:** el sistema puede responder correos con HITL, un test de runaway (loop artificial) es detenido por el kill switch, y una preferencia guardada en una sesión aparece en el recall de la siguiente.

### Fase 5: Ejecución aislada (semana 8)

- Pods Deno bajo demanda (pre-pull de imagen en bootstrap).
- Integración con Modal para escalado.
- Tool `runCode` con classification `confirm`.
- **Criterio de éxito:** puedes pedir "analiza este CSV con pandas", el sistema lo envía a Modal, obtiene el resultado, y te lo muestra.

### Fase 6: Interfaces avanzadas (semanas 9-10)

- Web Dashboard SPA con Monaco.
- Zone Widgets para comentarios inline.
- Iframes de applets (en `jinserver.com`).
- CLI con Ink.
- **Criterio de éxito:** puedes editar código en el dashboard, la IA comenta línea por línea, y aprobar cambios desde ahí.

### Fase 7: Producción y hardening (semanas 11-12)

- MCP servers para RAG de docs externas.
- Chaos tests.
- Documentación de operaciones (runbooks, incluido "VM perdida → reconstrucción desde IaC + backups").
- Auditoría de seguridad completa.
- **Criterio de éxito:** el sistema corre autónomamente por 7 días sin intervención manual salvo aprobaciones HITL.

---

## 15. Reglas de oro (nunca romper)

1. **jin-core jamás toca el socket de Docker/K3s.** Todo pasa por Executor.
2. **Ningún secreto en código, ni en env, ni en Git.** Todo en Infisical.
3. **Ningún backup no probado cuenta como backup.** Restore test mensual obligatorio.
4. **El HITL level de una tool jamás lo decide el LLM.** Es estático y solo cambiable con dual-confirm humano.
5. **Ningún modelo LLM hardcoded en código.** Todo por el registry `models.yaml`.
6. **Ningún input externo entra al prompt sin envolver en `<untrusted_content>`.**
7. **Toda tool destructiva es `dual-confirm`.** Cuando dudes entre `confirm` y `dual-confirm`, elige `dual-confirm`.
8. **Toda decisión mostrada al humano en HITL debe incluir la lista de inputs externos que influyeron.**
9. **El timeout nunca aprueba automáticamente.** Solo descarta o escala.
10. **Contenido generado por agentes se sirve solo desde el dominio sandbox (hoy `jinserver.com`), jamás desde el dominio de confianza (`jeanfranck.com` ni ninguno de sus subdominios).** La regla es por *rol*, no por hostname: si algún día cambian los dominios, lo que manda es la separación entre zona confiable y zona sandbox, no los nombres concretos.
11. **Ningún tipo se copia a mano entre repos.** Los contratos se generan desde OpenAPI.
12. **Cuando el blueprint y el código difieran, se corrige el código o el blueprint — nunca se ignora la diferencia.**
