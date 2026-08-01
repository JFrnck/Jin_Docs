# MODEL_ROUTING.md — Selección de modelos

> Guía definitiva de qué modelo usar para cada tipo de tarea, tanto en el sistema en runtime como en el desarrollo (Claude Code + Antigravity).
> Datos verificados: **Julio 2026**.

---

## 1. Familias disponibles y estado actual

### 1.1 Anthropic (Claude)

| Modelo           | Model ID           | Rol                         | Precio (in / out por 1M tok)           | Context |
| ---------------- | ------------------ | --------------------------- | -------------------------------------- | ------- |
| Claude Fable 5   | `claude-fable-5`   | Frontera (Mythos-class)     | $10 / $50                              | 1M      |
| Claude Opus 4.8  | `claude-opus-4-8`  | Flagship producción         | $15 / $75                              | 1M      |
| Claude Opus 4.7  | `claude-opus-4-7`  | Generación previa, aún útil | $15 / $75                              | 1M      |
| Claude Sonnet 5  | `claude-sonnet-5`  | Balanceado, default         | $3 / $15 (intro $2/$10 hasta Ago 2026) | 1M      |
| Claude Haiku 4.5 | `claude-haiku-4-5` | Rápido/barato               | $1 / $5                                | 200K    |

### 1.2 Google (Gemini)

| Modelo                      | Model ID                      | Rol                     | Precio (in / out por 1M tok) | Context |
| --------------------------- | ----------------------------- | ----------------------- | ---------------------------- | ------- |
| Gemini 3.5 Flash            | `gemini-3.5-flash`            | Frontera agentic        | $1.50 / $9                   | 1M      |
| Gemini 3.1 Pro              | `gemini-3.1-pro`              | Contexto masivo         | ~$1.25 / $10                 | **2M**  |
| Gemini 2.5 Flash-Lite       | `gemini-2.5-flash-lite`       | Barato en volumen       | $0.10 / $0.40                | 1M      |
| Antigravity Agent (preview) | `antigravity-preview-05-2026` | Agente autónomo managed | Uso por sandbox              | 1M      |

Notas:

- **Fable 5 y Mythos 5** son la clase superior a Opus. Solo úsalos si Opus 4.8 no basta.
- **Gemini 3.5 Flash** superó a 3.1 Pro en benchmarks agentic (Terminal-Bench, MCP Atlas) a menor costo, pero **3.1 Pro** sigue siendo el único con 2M de contexto.

---

## 2. Router del sistema en runtime

En `Jin_Core` (`src/model-provider/router.ts`) se define un mapa `TaskProfile → Model`. Este mapa NO se hardcodea — vive en `config/models.yaml`.

### 2.1 TaskProfiles definidos

```yaml
# config/models.yaml
profiles:
  reasoning_heavy:
    description: "Razonamiento complejo, debug, arquitectura"
    primary: claude-opus-4-8
    fallback: claude-sonnet-5
    max_tokens_input: 200000
    max_tokens_output: 8000
    temperature: 0.3

  coding_default:
    description: "Generación y edición de código diario"
    primary: claude-sonnet-5
    fallback: gemini-3.5-flash
    max_tokens_input: 100000
    max_tokens_output: 4000
    temperature: 0.2

  long_context:
    description: "Análisis de documentos largos, PDFs, hilos de correo"
    primary: gemini-3.1-pro
    fallback: claude-opus-4-8
    max_tokens_input: 1500000
    max_tokens_output: 8000
    temperature: 0.4

  extraction_fast:
    description: "Clasificación, extracción de entidades, resúmenes cortos"
    primary: claude-haiku-4-5
    fallback: gemini-2.5-flash-lite
    max_tokens_input: 32000
    max_tokens_output: 1000
    temperature: 0.1

  chat_conversational:
    description: "Chat con el usuario, respuestas conversacionales"
    primary: claude-sonnet-5
    fallback: claude-haiku-4-5
    max_tokens_input: 50000
    max_tokens_output: 2000
    temperature: 0.7

  code_execution_planner:
    description: "Genera código para pods Deno / Modal"
    primary: claude-opus-4-8
    fallback: claude-sonnet-5
    max_tokens_input: 100000
    max_tokens_output: 8000
    temperature: 0.2

  memory_consolidation:
    description: "Destila la sesión en entradas de memoria (sqlite-vec)"
    primary: claude-haiku-4-5
    fallback: gemini-2.5-flash-lite
    max_tokens_input: 64000
    max_tokens_output: 2000
    temperature: 0.2

  vision_analysis:
    description: "Análisis de screenshots, imágenes, gráficas"
    primary: claude-opus-4-8
    fallback: gemini-3.1-pro
    max_tokens_input: 100000
    max_tokens_output: 4000
    temperature: 0.3
```

### 2.2 Selector automático

`ModelProvider.selectModel(profile, hints)` acepta hints:

- `estimatedInputTokens` — si supera el límite del `primary`, cambia a `fallback` o al modelo con más contexto.
- `latencyRequirement: "low" | "normal"` — si `low`, baja al tier más rápido dentro del profile.
- `budgetRemaining` — si estás cerca del límite diario, degrada a modelos más baratos automáticamente.

### 2.3 Failover

Si el `primary` falla (rate limit, 5xx, timeout):

1. Reintenta 1 vez con backoff exponencial.
2. Cambia al `fallback` con logs de warning.
3. Registra métricas: `model_failover_total{from, to, reason}`.

---

## 3. Modelos para desarrollo (IDEs)

### 3.1 Claude Code

| Situación                                             | Modelo                      | Notas                                     |
| ----------------------------------------------------- | --------------------------- | ----------------------------------------- |
| Trabajo diario, features rutinarias                   | **Sonnet 5**                | Default. 90%+ del tiempo                  |
| Bugfix simple                                         | **Sonnet 5**                |                                           |
| Refactor con muchos archivos                          | **Sonnet 5** o **Opus 4.8** | Opus si involucra decisiones no triviales |
| Diseño de arquitectura, ADRs                          | **Opus 4.8**                | Siempre                                   |
| Debug de bug sutil que llevó >30 min sin progreso     | **Opus 4.8**                | Cambia proactivamente                     |
| Revisión de seguridad (HITL, audit, budget, security) | **Opus 4.8**                | Siempre                                   |
| Escribir tests, docs                                  | **Sonnet 5**                | Suficiente                                |
| Preguntas rápidas ("cómo se hace X en NestJS")        | **Haiku 4.5**               | Rápido y barato                           |

**Regla de dedo:** empieza en **Sonnet 5**. Cambia a **Opus 4.8** cuando el problema resista.

### 3.2 Antigravity

| Situación                                       | Modelo                          | Notas                                         |
| ----------------------------------------------- | ------------------------------- | --------------------------------------------- |
| Trabajo autónomo largo, scaffolding             | **Gemini 3.5 Flash**            | Default en Antigravity                        |
| Manipular codebase con muchos archivos          | **Gemini 3.5 Flash**            | Rápido y competente                           |
| Contexto >500k tokens (analizar todo un módulo) | **Gemini 3.1 Pro**              | Único con 2M                                  |
| Análisis de PDF grande de Canvas                | **Gemini 3.1 Pro**              | Multimodal + contexto largo                   |
| Tarea multi-paso autónoma en sandbox            | **antigravity-preview-05-2026** | Es el agente managed, tiene sandbox integrado |
| Preguntas rápidas                               | **Gemini 2.5 Flash-Lite**       | Mínimo costo                                  |

---

## 4. Decisión rápida: ¿qué modelo uso ahora mismo?

```
¿Es tarea de runtime del sistema (agente resolviendo algo para el usuario)?
├─ Sí → usa el TaskProfile en `models.yaml` que corresponda
└─ No, es desarrollo
    ├─ Estoy en Claude Code
    │   ├─ ¿Involucra seguridad, HITL, audit, budget? → Opus 4.8
    │   ├─ ¿Diseño no trivial? → Opus 4.8
    │   ├─ ¿Debug largo sin progreso? → Opus 4.8
    │   ├─ ¿Pregunta trivial? → Haiku 4.5
    │   └─ Default → Sonnet 5
    └─ Estoy en Antigravity
        ├─ ¿Contexto >500k? → Gemini 3.1 Pro
        ├─ ¿Análisis de PDF/imagen? → Gemini 3.1 Pro
        ├─ ¿Autónomo multi-paso? → Antigravity Agent
        ├─ ¿Pregunta trivial? → Gemini 2.5 Flash-Lite
        └─ Default → Gemini 3.5 Flash
```

---

## 5. Costos estimados mensuales (uso personal)

Basado en uso proyectado del owner:

- ~30 tareas ligeras/día × 30 días = 900 llamadas Sonnet 5 ≈ **$5-8/mes**
- ~10 tareas pesadas/día × 30 días = 300 llamadas Opus 4.8 ≈ **$20-30/mes**
- ~2 análisis largos/día × 30 días = 60 llamadas Gemini 3.1 Pro ≈ **$5-10/mes**
- ~50 tareas triviales/día × 30 días = 1500 llamadas Haiku/Flash-Lite ≈ **$1-3/mes**

**Total estimado LLM APIs: $30-50/mes** (con budget guard evitando runaways).

Presupuesto duro diario en `config/budget.yaml`: **$10/día**.

---

## 6. Reglas anti-obsolescencia

### 6.1 Nunca hardcoded

Ni en código de producción, ni en scripts, ni en tests salvo mocks. Todo modelo pasa por el `ModelProvider`.

### 6.2 Verificación mensual

Cron mensual que llama a los endpoints de listado de modelos de Anthropic y Google (`/v1/models`) y compara con `models.yaml`. Si hay nuevos modelos o alguno está deprecado → notifica Telegram.

### 6.3 Actualización de este documento

Cada vez que se actualice `models.yaml` con un modelo nuevo, se actualiza este documento en el mismo PR. Sin excepciones.

### 6.4 Deprecation gracia

Cuando Anthropic o Google anuncian deprecation de un modelo:

1. Se marca en `models.yaml` como `deprecated: true`.
2. Se cambia el `primary` de todos los profiles que lo usan.
3. Se ejecuta un test regression con el nuevo modelo antes de mergear.

---

## 7. MCP servers y RAG externo

Para documentación técnica externa (React, Tailwind, NestJS, Prisma, etc.), NO usamos scraping propio. Conectamos MCP servers oficiales:

- **Context7** — documentación general de librerías.
- **gemini-interactions-api Skill** — para Antigravity específicamente.
- MCPs oficiales de librerías que los publiquen (ej. Prisma, Anthropic).

El agente hace tool-calling a estos MCPs on-demand cuando necesita documentación fresca. Esto reemplaza completamente el pipeline Jina/Firecrawl del blueprint original.

Nota de frontera (ver BLUEPRINT 6.4): MCP = conocimiento externo fresco · pgvector = corpus propio indexado · sqlite-vec = memoria interna del agente. Tres problemas, tres herramientas.

## 7.1 Contexto de versiones automático (anti-alucinación)

Todo prompt del `ModelProvider` que involucre generar código sobre librerías del stack debe incluir automáticamente las versiones exactas del `package.json` como contexto.

### Implementación

`Jin_Core/src/config/versions.ts`:

```typescript
export async function getInstalledVersions(packageNames: string[]): Promise<Record<string, string>>;
```

Lee el `package.json` **del propio repo** y devuelve un mapa `{ nombre: version }`. (Ya no hay monorepo: cada repo conoce solo sus versiones, que son las únicas que necesita. Cuando un IDE trabaja en otro repo, lee el `package.json` de ese repo.)

### Uso en el ModelProvider

Antes de cada llamada LLM con tarea de código, el ModelProvider prepend al system prompt:

Estás trabajando con estas versiones exactas del stack:

@nestjs/core: {version}
react: {version}
vite: {version}
@prisma/client: {version}
zod: {version}

Consulta Context7 MCP con IDs de versión específicos cuando necesites documentación. Si Context7 no tiene información para una versión exacta, avisa explícitamente y no inventes. No uses APIs de versiones anteriores aunque las conozcas; asumí que hubo cambios.

### Razón (pendiente de ADR — ver STATUS.md)

> Nota: esta sección citaba "ver ADR 0002" en el bundle de docs original, antes de que existiera ningún ADR — colisión de numeración con `docs/adr/0002-audit-log-request-id.md`, que trata un tema no relacionado. `versions.ts` y este mecanismo de anti-alucinación todavía no están implementados (quedaron fuera de alcance de Fase 3.1 porque solo aplican a tareas de código, y Canvas/shadowing usa el profile `long_context`, no uno de código) — se escribirá el ADR real cuando se construyan.

Entre versiones menores de frameworks recientes hubo cambios sutiles de API que los LLMs actuales pueden no conocer bien. Anclar el prompt a versiones exactas + forzar al LLM a consultar Context7 con esas versiones reduce alucinaciones sutiles del tipo "método que existía en v11.0 pero se renombró en v11.1".

### Frecuencia de actualización

Automático — el archivo se lee del disco en cada request. Cuando `pnpm update` cambia una versión en el repo, el próximo prompt refleja el cambio sin editar código.
