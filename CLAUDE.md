# CLAUDE.md — Directivas específicas para Claude Code

> Este archivo lo carga Claude Code automáticamente. Extiende (no reemplaza) `AGENTS.md`.
> **Lee `AGENTS.md` primero.** Todo lo declarado allí aplica. Aquí solo va lo específico a Claude Code.
> Workspace local: carpeta `Jin/` con todos los repos clonados lado a lado (ver `AGENTS.md` 4.1).

---

## 1. Selección de modelo

### 1.1 Reglas por tipo de tarea

| Tarea                                                  | Modelo recomendado | Cuándo escalar a Opus |
| ------------------------------------------------------ | ------------------ | --------------------- |
| Refactor rutinario, features pequeñas, bugfixes obvios | **Sonnet 5**       | Nunca — Sonnet basta  |
| Escribir tests, docs, migrations                       | **Sonnet 5**       | Nunca                 |
| Diseño de arquitectura, decisiones no triviales        | **Opus 4.8**       | Default aquí          |
| Debug de bugs sutiles (race conditions, memory leaks)  | **Opus 4.8**       | Default aquí          |
| Auditoría de seguridad, revisión de HITL/audit/secrets | **Opus 4.8**       | Default aquí          |
| Preguntas rápidas, "cómo se hace X"                    | **Haiku 4.5**      | Nunca                 |

Si dudas: **Sonnet 5**. Si el problema requirió >30 min sin progreso: cambia a **Opus 4.8**.

### 1.2 Extended thinking

- Actívalo (`thinking: enabled`) para:
  - Bugs no reproducibles.
  - Diseños de arquitectura.
  - Análisis de logs complejos.
  - Tareas de HITL classifier y audit log.
- Déjalo apagado para tareas mecánicas (crear boilerplate, escribir tests obvios).

---

## 2. Comportamiento en Claude Code

### 2.1 Planning mode

Actívalo (Shift+Tab dos veces en la terminal) para:

- Cualquier cambio que toque >5 archivos.
- Cualquier cambio en `Jin_Core/src/hitl/`, `Jin_Core/src/audit/`, `Jin_Core/src/memory/`, o el repo `Jin_Executor`.
- Cualquier cambio en manifests de infra (repo `Jin_Infra`).
- Cualquier cambio de dependencias en el `package.json` de cualquier repo.
- Cualquier cambio que rompa o modifique un contrato OpenAPI.

Cuando el plan esté listo, léelo con el owner línea por línea antes de aceptar.

### 2.2 Tool use

- **Read** antes de **Edit**. Nunca edites un archivo que no acabas de leer.
- **Grep/Glob** antes de asumir dónde vive el código. Recuerda que el workspace tiene 6 repos: verifica en cuál estás antes de editar.
- **Bash** para: linting, tests, git ops. Nunca para cambios destructivos sin confirmación. Cuidado: cada repo tiene su propio `.nvmrc` — usa `nvm use` al cambiar de repo.

### 2.3 Sub-agentes

- Usa `Task` con sub-agentes para búsquedas paralelas ("busca todos los usos de X y también todos los usos de Y").
- No los uses para tareas secuenciales — se pierden en contexto.
- Límite: 3 sub-agentes concurrentes por sesión.

---

## 3. Áreas de responsabilidad principal

Claude Code lidera estas partes (ver `docs/WORKFLOW.md` para el mapa completo):

- Repo `Jin_Executor` **completo** — RBAC, ejecución aislada, ciclo de vida de pods.
- Repo `Jin_Web` **completo** (desde 2026-08-02, decisión explícita del owner — no es el criterio habitual de "dividir por fortaleza", ver `docs/WORKFLOW.md` §2.1) — dashboard React, incluye Fases 6.2 y 6.3 del roadmap.
- En `Jin_Core`:
  - `src/hitl/**` — clasificador HITL, timeouts, aprobaciones.
  - `src/audit/**` — audit log, hash chain, verificación de integridad.
  - `src/budget/**` — budget guard, kill switch, rate limiter.
  - `src/security/**` — injection sanitizer, whitelists, RBAC validator.
  - `src/memory/**` — memoria extendida sqlite-vec.
  - `src/model-provider/**` — router de LLMs.
- Tests de las piezas anteriores.
- ADRs (Architecture Decision Records) transversales en `Jin_Docs/docs/adr/`.

---

## 4. Comandos frecuentes

> **Nota:** algunos comandos solo existen a partir de fases posteriores del roadmap (ver `docs/BLUEPRINT.md` sección 14). Antes de correr uno, verifica que esté en el `package.json` del repo. Cada comando se corre **dentro del repo correspondiente** — ya no hay `--filter` ni pipeline global.

```bash
# --- En cada repo Node (core, executor, web, cli) ---
nvm use                            # Respeta el .nvmrc del repo
pnpm install
pnpm dev                           # Hot reload del repo actual
pnpm typecheck
pnpm lint
pnpm test

# --- Contratos (core, web, cli) ---
pnpm generate:api                  # Regenera tipos desde el openapi.json del productor
pnpm generate:contract             # (core/executor) Regenera contracts/openapi.json propio

# --- Jin_Core desde Fase 2 ---
pnpm test:integration              # Integration tests con testcontainers
pnpm test:coverage
pnpm db:migrate                    # Aplicar migraciones
pnpm db:migrate:down               # Rollback última migración

# --- Jin_Web desde Fase 6 ---
pnpm test:e2e                      # E2E con Playwright

# --- Levantar el stack local completo ---
# Cada repo corre su propio `pnpm dev` en su terminal (core + executor mínimo).
# Postgres/Redis locales: docker compose -f Jin_Infra/docker-compose.dev.yaml up
```

---

## 5. Estilo de comunicación en la sesión

- **Español por defecto** (owner escribe en español). Cambiar a inglés solo si el owner lo pide o si es contenido técnico donde el inglés es estándar (nombres de funciones, mensajes de commit).
- **Directo, sin flattery**. No abras respuestas con "great question" o similares.
- **Explica el porqué** de cada decisión no obvia.
- **Cuando no estás seguro, dilo.** "Creo que X, pero verifica con tests" es mejor que afirmarlo con confianza fingida.

---

## 6. Coordinación con Antigravity

Antes de empezar cualquier trabajo:

1. Lee `Jin/Jin_Docs/STATUS.md` para ver qué está haciendo Antigravity ahora y en qué repo.
2. Verifica que tu rama no colisiona con `feature/antigravity/*` activas **en el mismo repo** (repos distintos = cero colisión posible).
3. Si tu tarea toca archivos de core que Antigravity está editando, **detente** y coordina con el owner.

Al terminar tu sesión:

1. Actualiza `STATUS.md` marcando tu trabajo como completo o en pausa.
2. Deja tu rama en estado limpio (commits pushed, sin cambios uncommitted).

Ver `docs/WORKFLOW.md` para el protocolo completo.

---

## 7. Cuando algo requiere criterio del owner

Situaciones donde debes preguntar en lugar de decidir:

- El blueprint no cubre el caso.
- Hay 2+ formas razonables y ninguna es claramente mejor.
- Un cambio afecta la seguridad, HITL o audit log.
- Necesitas introducir una librería nueva.
- Un cambio de API rompe el contrato OpenAPI que otro repo consume.
- Un test que "debería fallar" está pasando.
- Un test que "debería pasar" está fallando por razones que no entiendes.

Formato de pregunta:

> Contexto: [1-2 líneas]
> Opciones: [A, B, C con trade-offs]
> Mi recomendación: [una, con razón]
> ¿Sigo?
