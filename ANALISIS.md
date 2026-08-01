# ANALISIS.md — Refinamiento JI-PYEONG → Jin

> Análisis y decisiones del refinamiento (2026-07-19). Los cambios aceptados ya están aplicados en los demás docs. Lo que aquí es solo opinión está marcado como **[recomendación, no aplicada]**.

---

## 1. Nombre

- **Proyecto:** Jin (nombre, branding, CLI, repos).
- Historia del nombre: JI-PYEONG → Yormungander/Yormun (referencia a la serpiente nórdica Jörmungandr) → **Jin** (2026-07-31, decisión del owner). El nombre corto ganó por ergonomía: se escribe todos los días en repos, paquetes y namespaces.
- Renombres derivados: repos `Jin_*` (carpeta local `Jin/`), namespaces K8s en minúscula `jin` / `jin-executor` (K8s lo exige), clase base de errores `JinError`, bucket R2 `jin-backups`.

## 2. Dominios: cuál para qué

| Dominio | Rol | Hosts |
| --- | --- | --- |
| **jeanfranck.com** | **Identidad del owner.** Portafolio personal, no forma parte de Jin. | Raíz y paths (`/proyecto`, etc.) |
| **jin.jeanfranck.com** | **Núcleo confiable.** Lo que tú usas a diario. | Dashboard (Cloudflare Pages) + API bajo el path `/api`, ruteada al túnel |
| **jinserver.com** | **Zona sandbox / no-confiable.** Todo lo que generan los agentes. | `<slug>.jinserver.com` — applets, previews, share links, con TTL obligatorio. Raíz sin contenido, a propósito. |

Razones:

1. **Aislamiento de origen (la razón técnica fuerte).** HTML/JS generado por LLM nunca debe compartir dominio registrable con el dashboard donde vive tu sesión: cookies con scope al dominio padre, y phishing de subdominios parecidos, quedan neutralizados por diseño. Es el mismo patrón de GitHub (`githubusercontent.com`) y Google (`googleusercontent.com`).
2. **Ergonomía.** El corto lo escribes tú todos los días; el largo se lo comen los agentes y los UUIDs.
3. Ambos con wildcard cert (cert-manager DNS-01), ambos detrás del mismo Cloudflare Tunnel. Cloudflare Access obligatorio en `dash` y `grafana`, y también en los applets (ya estaba en el blueprint).

## 3. Función de cada app y veredicto

| Repo | Antes | Función | Veredicto |
| --- | --- | --- | --- |
| **Jin_Core** | `apps/ji-pyeong` | El cerebro: API REST/WS, auth, crons, BullMQ, HITL, audit, budget, security, model-provider, memoria (sqlite-vec), integraciones, y el bot de Telegram **como módulo interno** (decisión original correcta: necesita acceso directo a HITL/audit sin latencia de red). | Se queda. Es la única app "grande". |
| **Jin_Executor** | `apps/executor` | Único proceso autorizado a hablar con K8s. Crea/destruye pods en `agents-sandbox`, decide local vs Modal. | Se queda **separado sí o sí**: es la frontera de seguridad central del proyecto. Fusionarlo con core rompería la regla de oro #1. |
| **Jin_Web** | `apps/web` | Dashboard React (Monaco, applets, aprobaciones). Se despliega en Cloudflare Pages, ni siquiera corre en tu VM. | Se queda, y es el **mayor beneficiado del split**: árbol de dependencias enorme y frágil (Monaco, React 19, Vite) que era el candidato #1 a romper el CI de todos. Ciclo de vida y destino de deploy distintos = repo distinto. |
| **Jin_CLI** | `apps/cli` | Cliente Ink del mismo API. Admin/debug. | Se queda como repo aparte pero es opcional y de baja prioridad (Fase 6). Cliente delgado: no comparte código con nadie, solo consume el API. |
| **Jin_Infra** | `infra/` + repo GitOps | Manifests Kustomize, Flux, bootstrap, scripts de backup. | Ya era repo separado en el blueprint (12.1). Ahora también absorbe `scripts/backup`. Los Dockerfiles se van a cada repo de app (cada CI construye su imagen). |
| **Jin_Docs** | `docs/` + raíz | BLUEPRINT, WORKFLOW, MODEL_ROUTING, PROMPTS, ADRs transversales, `STATUS.md`. | Nuevo meta-repo. Es el punto de coordinación entre IDEs. |
| ~~packages/shared-*~~ | `shared-types`, `shared-config`, `shared-audit` | — | **Se disuelven.** Eran la razón de ser del monorepo. `shared-types` → tipos generados desde OpenAPI; `shared-config` → cada repo valida su config con Zod localmente; `shared-audit` → solo lo usaba core, se pliega a `core/src/audit`. |

## 4. Salida del monorepo: opciones y decisión

Tu dolor real: una dependencia rota en una app tumbaba el CI de todo, y una sola versión de Node forzada para apps con necesidades distintas. Ambos son síntomas del acoplamiento por workspaces (lockfile único, hoisting, pipeline compartido).

### Opciones

**A. Multi-repo + contratos OpenAPI — la elegida.**
Un repo por app. Nada de paquetes compartidos: core y executor generan `contracts/openapi.json` desde su propio código (`@nestjs/swagger`); web y cli generan tipos con `openapi-typescript` (`pnpm generate:api`). El contrato nace del runtime real, así que no puede desincronizarse en silencio: si core cambia el API, el `tsc` del consumidor falla al regenerar — el drift se detecta, no se hereda.

- Pros: CI, deps, lockfile, Node y deploy 100% independientes por repo (exactamente lo que pediste: mantenimiento y escalado unitario). Cero publishing. Radio de explosión de cualquier fallo = 1 repo.
- Contras: un cambio cross-cutting (API + web) son 2 PRs; cambios de API deben ser retrocompatibles o hacerse en dos pasos (expand → contract). Con un solo dev y contratos generados, es un costo bajo.

**B. Multi-repo + paquete `@jin/contracts` en GitHub Packages.**
Tipos centralizados escritos a mano y publicados como npm privado.

- Pros: un solo lugar para los tipos.
- Contras: cada cambio = bump + publish + actualizar N repos; reintroduce el acoplamiento que te dolía, en cámara lenta. Solo tendría sentido si apareciera *lógica* compartida real (no solo tipos). Hoy no existe.

**C. Un repo, apps desacopladas (sin workspaces).**
Carpetas con `package.json`, lockfile y `.nvmrc` propios; CI filtrado por paths.

- Pros: resuelve la rotura cruzada conservando un solo clone.
- Contras: historia y PRs mezclados, Flux vigilando un repo con ruido de apps, y no da el mantenimiento/escalado por unidad que quieres. Descartada por tu propio criterio.

**D. Submodules / meta-repo git.** Dolor crónico de sincronización. Descartada sin debate.

### Cómo queda (opción A)

```
Jin/              ← carpeta local, no es un repo
  Jin_Docs/
  Jin_Core/       Node 24 LTS
  Jin_Executor/   Node 24 LTS
  Jin_Web/        Node según lo que pida Vite/React (libre)
  Jin_CLI/        Node 24 LTS
  Jin_Infra/      (sin Node: YAML + bash)
```

Detalle importante: **la comodidad de "todo en una carpeta" no se pierde.** Clonas los seis repos bajo `Jin/` y abres esa carpeta en el IDE: Claude Code y Antigravity ven todo el contexto, como antes. Lo que muere es el acoplamiento de CI/deps, no la vista unificada local.

Node: `.nvmrc` + `engines` por repo; `setup-node` en CI lee `.nvmrc`. Core/executor en **Node 24** (LTS activo desde Oct 2025; el 22 del blueprint ya está en maintenance — probable causa de tus errores de versión). Web puede moverse a su ritmo sin arrastrar al backend.

Mitigación del único costo real (boilerplate ×6): workflows de CI reutilizables (`workflow_call`) viviendo en Jin_Infra, y un template repo para futuros servicios.

## 5. Tecnología definida (por repo)

| Repo | Stack | Deploy |
| --- | --- | --- |
| Jin_Core | Node 24, TS strict, NestJS, BullMQ, Zod, pino, grammY, `@nestjs/swagger`, Drizzle **[recomendación]** o Prisma, pgvector, better-sqlite3 + sqlite-vec | Imagen → GHCR → Flux → K3s |
| Jin_Executor | Node 24, TS strict, NestJS, `@kubernetes/client-node`, Modal SDK, Zod | Igual |
| Jin_Web | Vite, React 19, React Router v7, Monaco, tipos generados de OpenAPI | Cloudflare Pages |
| Jin_CLI | Node 24, Ink, tipos generados de OpenAPI | npm local / binario |
| Jin_Infra | Kustomize, Flux, cert-manager, kube-linter en CI, bash | `git push` → Flux |
| Pods efímeros | Deno 2.x | Creados por executor |

Sobre ORM: el blueprint dejaba "Prisma o Drizzle" abierto. **[Recomendación]** Drizzle: más liviano en RAM/arranque sobre ARM, SQL-first (mejor con pgvector crudo), sin engine binario. Cualquiera de los dos es defendible; decídelo en la Fase 2.1 y escribe el ADR.

## 6. sqlite-vec: memoria extendida local — aceptado, así encaja

**Qué es cada cosa (frontera clara):**

- **pgvector** (se queda): el *corpus* — correos indexados, PDFs de Canvas, notas. Grande, con JOINs relacionales contra `tasks` y el audit log. Su justificación original sigue intacta.
- **sqlite-vec** (nuevo): la *memoria del agente* — hechos sobre ti, preferencias, resúmenes episódicos de sesiones, decisiones tomadas y lecciones. Pequeña (≤100k vectores), curada, caliente: se lee al inicio de cada sesión de agente (recall) y se escribe al cierre (consolidación).

**Por qué en SQLite y no en más tablas de pgvector:** un archivo `memory.db` en el volumen de core: cero infra adicional, sobrevive y se restaura independiente de Postgres, se copia a tu laptop con un `cp`, y su backup es trivial (entra al cron nocturno: `.backup` → age → R2). A esa escala, KNN brute-force responde en milisegundos.

**Estado verificado (jul 2026):** sqlite-vec ya soporta columnas de metadata con filtrado (el mayor update desde v0.1) y tiene índices ANN en alpha (rescore, IVF experimental, DiskANN). Conclusión: **usa brute-force + filtros de metadata, que es lo estable; no dependas del ANN todavía.** A tu escala no lo necesitas.

**Reglas de diseño aplicadas:** solo `Jin_Core` (módulo `src/memory/`) toca `memory.db`; los pods jamás. WAL mode, un solo writer. Metadata mínima por entrada: `tipo`, `fuente`, `fecha`, `modelo_embedding` (esto último permite migrar de modelo de embeddings sin adivinar). Si algún día crece de más, migrar a pgvector es un script.

**MCP para docs externas: se mantiene tal cual** (Context7 + MCPs oficiales). Correcta división: MCP = conocimiento *externo* fresco on-demand; sqlite-vec = memoria *interna* del sistema; pgvector = corpus *propio* indexado. Tres problemas, tres herramientas, cero solapamiento.

## 7. Pods siempre encendidos: fuera — aplicado

Tu argumento es correcto y lo hago explícito: **en un flujo con HITL, la latencia dominante es humana.** Un warm pool ahorra ~3 s de arranque en tareas donde tú tardas segundos o minutos en aprobar. Para un usuario, es optimizar lo que no importa.

Cambios aplicados:

- Warm pool eliminado. El executor crea el pod **bajo demanda**, ejecuta, destruye. Con la imagen Deno pre-descargada en el nodo (pre-pull en bootstrap), el arranque es ~2-5 s.
- La ResourceQuota de `agents-sandbox` (6 Gi / 1500m) pasa de reserva ocupada a **techo de ráfaga**: consumo idle cero.
- Executor más simple: muere la lógica de replenish/readiness del pool (menos código crítico que testear).
- **Coherencia aplicada también a core:** 2 réplicas "para rolling updates" eran otro pod siempre encendido redundante para un solo usuario. Ahora **1 réplica** con `maxSurge: 1, maxUnavailable: 0` — el rolling update sigue siendo sin downtime (K8s levanta la nueva antes de matar la vieja). Ahorro: ~512Mi-1Gi. Si no estás de acuerdo, es revertir un número.

## 8. Pros y contras del proyecto (análisis pedido, sin cambios aplicados)

### Pros

1. **Seguridad fuera de serie para un proyecto personal.** HITL de 4 niveles con dual-confirm temporizado, audit log con hash chain, separación core/executor con RBAC mínimo, egress whitelist por tool, sanitizador de injection con nonce por sesión. Mejor que lo que tienen muchas startups. Es la parte del blueprint que no tocaría ni una coma.
2. **Disciplina de costos real:** free tiers + budget guard de 3 niveles + kill switch + estimación honesta ($30-50/mes LLM). Pocas veces se ve un "personal OS" con contabilidad.
3. **Anti-obsolescencia pensada:** registry de modelos sin hardcode, MCP para docs frescas, prompts anclados a versiones exactas del package.json. Envejece bien por diseño.
4. **Cultura de ingeniería:** docs-first, ADRs, backups *probados* mensualmente, TDD en lo crítico. La filosofía "lo aburrido y crítico antes que lo divertido" es la correcta y es rara.
5. El multi-repo ahora **alinea la estructura del código con la filosofía de blast-radius** que la seguridad ya tenía: un fallo en web no puede tocar a core, igual que un pod no puede tocar a Postgres.

### Contras y riesgos

1. **Complejidad de plataforma alta para 1 usuario.** K3s + Flux + Traefik + cert-manager + Prometheus + Loki + Grafana + Tempo + Infisical + PgBouncer ≈ 12 componentes operados por ti, para ~50 tareas/día. Cada uno es algo que se actualiza y se rompe. ~15% de la RAM se va en observabilidad. **[Recomendación, no aplicada]:** K3s se queda — el modelo de seguridad (namespaces, NetworkPolicies, RBAC) está construido sobre él y es la columna vertebral. Pero recorta la periferia al arrancar: pospón Tempo (tracing) y PgBouncer hasta que duelan; Prometheus+Loki+Grafana mínimos bastan las primeras 8 semanas.
2. **Una sola VM = punto único de fallo.** Aceptado explícitamente en no-objetivos, y los backups probados lo mitigan. Riesgo residual real: Oracle es conocido por reclamar instancias Always Free con poco uso. Mitigación ya implícita: todo es IaC + backups → reconstruible en horas. Ten el runbook de "VM perdida" escrito (Fase 7).
3. **Dependencia de free tiers ajenos** (OCI, Modal $30, R2, Pages). Las políticas cambian sin avisarte. El diseño ya lo abstrae razonablemente (Executor decide local vs remoto; R2 es S3-compatible).
4. **Roadmap de 12 semanas es ambicioso** junto a una mudanza a Canadá y estudios. Las fases 1-2 (infra + núcleo con HITL/audit) son el 80% del valor; si algo se recorta, que sea del final (Monaco, applets), no del principio. La filosofía del README ya lo dice — respétala cuando llegue la tentación.
5. **Costo del multi-repo:** cambios cross-cutting = varios PRs, y disciplina de retrocompatibilidad en el API. Bajo, pero existe; los contratos OpenAPI lo convierten de "riesgo silencioso" a "error de compilación".
6. **El riesgo humano del HITL:** el sistema anti-"aprobar por reflejo" (30 s) está bien diseñado, pero con un solo aprobador cansado sigue siendo el eslabón débil. Los 30 segundos no son negociables contigo mismo.

### Conclusión

El proyecto es sólido y está sobre-ingenierizado *a propósito* donde debe (seguridad) y un poco donde no aporta (plataforma/observabilidad, ya señalado). Los cambios de hoy — multi-repo con contratos generados, sqlite-vec como memoria, pods bajo demanda, dominios con aislamiento de origen — lo hacen más simple de operar sin ceder nada de la seguridad. El riesgo #1 del proyecto no es técnico: es que las 12 semanas compitan con tu vida real. Protege las fases 1-2 y el resto puede esperarte.

---

## 9. Registro de cambios aplicados a los docs

1. Nombre JI-PYEONG → Jin en todos los docs, clases, namespaces, buckets.
2. Sección de dominios reescrita: jin.jeanfranck.com (confiable) / jinserver.com (sandbox).
3. Monorepo eliminado: multi-repo A con contratos OpenAPI; layout, ownership, comandos, prompts y CI actualizados; packages `shared-*` disueltos.
4. sqlite-vec añadido (BLUEPRINT 3.3.1 + módulo `memory` + backup nocturno + roadmap Fase 4).
5. Warm pool eliminado; pods bajo demanda; quota como techo; core a 1 réplica.
6. Node 22 → **Node 24 LTS** por repo (`.nvmrc`).
7. MCP (Context7 etc.) intacto, reafirmado.
8. Todo lo demás (HITL, audit, budget, K3s, observabilidad, roadmap) **sin cambios**: las opiniones quedaron en este doc como recomendaciones.

Fuentes: [sqlite-vec releases](https://github.com/asg017/sqlite-vec/releases) · [sqlite-vec metadata](https://alexgarcia.xyz/blog/2024/sqlite-vec-metadata-release/index.html) · [Node.js releases](https://nodejs.org/en/about/previous-releases) · [endoflife.date/nodejs](https://endoflife.date/nodejs)
