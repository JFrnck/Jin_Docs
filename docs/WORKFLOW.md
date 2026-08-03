# WORKFLOW.md — Coordinación entre Antigravity IDE y Claude Code

> **Objetivo:** que dos LLMs distintos trabajen sobre el mismo proyecto sin pisarse ni introducir conflictos silenciosos.
> Regla mental: **piensa en cada IDE como un colaborador humano remoto**. Aplican las mismas reglas de coordinación: ownership, branches, handoffs, no editar lo mismo al mismo tiempo.
> Con el multi-repo, la mayor parte de la coordinación se resuelve sola: **repos distintos = colisión imposible.** El único territorio compartido real es `Jin_Core`.

---

## 1. Filosofía de la división

No es "dividir por tarea" sino "dividir por fortaleza". Cada IDE tiene un carácter distinto:

**Claude Code** brilla en:

- Razonamiento cuidadoso sobre lógica compleja.
- Refactors quirúrgicos con muchos archivos pequeños.
- TDD estricto.
- Seguridad, auditoría, verificación.
- Debugging de bugs sutiles.

**Antigravity** (Gemini agents) brilla en:

- Trabajo autónomo multi-paso con contexto grande.
- Scaffolding de proyectos y estructuras.
- Manipulación de muchos archivos a la vez (gracias a los 2M de contexto de Gemini 3.1 Pro).
- Integraciones con navegador (útil para probar el frontend).
- Migraciones y transformaciones sistemáticas.

Usa cada uno donde su fortaleza importa.

---

## 2. Mapa de ownership (autoritativo)

### 2.1 Por repo

| Repo                | Owner primario | Razón                                                            |
| ------------------- | -------------- | ---------------------------------------------------------------- |
| **Jin_Executor** | Claude Code    | Frontera de seguridad: RBAC, ejecución aislada                   |
| **Jin_Web**      | Claude Code    | Decisión explícita del owner (2026-08-02), no fortaleza técnica — ver nota abajo |
| **Jin_CLI**      | Antigravity    | Ink CLI, scaffolding rápido                                      |
| **Jin_Infra**    | Antigravity    | Manifests Kustomize, muchos archivos similares                   |
| **Jin_Docs**     | Compartido     | Cambios por PR con revisión del owner                            |
| **Jin_Core**     | Compartido     | Único repo donde ambos trabajan — ver mapa interno en 2.2        |

El owner primario de un repo puede trabajar sin avisar. El otro IDE solo entra a ese repo con negociación previa (nota en `STATUS.md` + ok del owner humano).

> **Excepción al criterio "dividir por fortaleza" (sección 1):** `Jin_Web` pasó de Antigravity a Claude Code el 2026-08-02 por decisión explícita del owner ("confío más en ti con lo que respecta a web"), no por un cambio en las fortalezas relativas de cada IDE. Ver `STATUS.md` sección "Decisiones del owner" para el detalle. El resto del mapa no cambia: `Jin_CLI`/`Jin_Infra` siguen con Antigravity.

### 2.2 Dentro de Jin_Core (el único territorio mixto)

**Claude Code lidera:**

| Ruta                     | Razón                            |
| ------------------------ | -------------------------------- |
| `src/hitl/**`            | Lógica crítica de seguridad      |
| `src/audit/**`           | Hash chain sagrado               |
| `src/budget/**`          | Kill switch, presupuesto         |
| `src/security/**`        | Sanitizer, whitelists            |
| `src/memory/**`          | Memoria sqlite-vec               |
| `src/model-provider/**`  | Router de LLMs                   |
| `src/tools/registry.ts`  | Declaración de hitlLevel         |
| Tests de las rutas anteriores | Coherencia con la implementación |

**Antigravity lidera:**

| Ruta                     | Razón                                      |
| ------------------------ | ------------------------------------------ |
| `src/integrations/**`    | Boilerplate de SDKs (Canvas, Google, etc.) |
| `src/telegram/**`        | Scaffolding del bot (la lógica HITL que consume es de Claude) |
| Migrations de DB         | Muchas migrations pequeñas                 |

**Compartido dentro de core (ping antes de editar):** `package.json`, `tsconfig.json`, `.github/workflows/**`, `config/*.yaml`, `contracts/`.

---

## 3. Ramas y convenciones

### 3.1 Naming (igual en todos los repos)

- `feature/claude/<slug>` — trabajo de Claude Code.
- `feature/antigravity/<slug>` — trabajo de Antigravity.
- `fix/<slug>` — bugfixes (cualquier IDE puede tomarlos en sus repos).

### 3.2 Ciclo de vida

1. Crear rama desde `main` del repo correspondiente.
2. Trabajar exclusivamente en tu rama.
3. Push frecuente (al menos al final de cada sesión).
4. PR a `main` cuando esté listo.
5. Solo humano hace el merge final (nunca auto-merge en piezas críticas).

### 3.3 Un IDE, una rama activa por repo

No abras dos ramas en el mismo repo a la vez. Si necesitas cambiar de contexto, commitea o stash, cambia de rama, y cuando vuelvas revisa `STATUS.md`.

### 3.4 Cambios cross-repo

Cuando una feature toca productor y consumidor (ej. endpoint nuevo en core + pantalla en web):

1. Primero el PR del productor (core), con el cambio **retrocompatible**.
2. Merge + regeneración del contrato (`contracts/openapi.json`).
3. Después el PR del consumidor (web), regenerando tipos (`pnpm generate:api`).
4. Nunca al revés, y nunca un breaking change directo — usa expand → contract.

---

## 4. STATUS.md — mecanismo de coordinación ligero

Vive en la raíz del repo `Jin_Docs` (es decir, `Jin/Jin_Docs/STATUS.md`). Ambos IDEs lo leen al empezar y lo actualizan al terminar. Cubre los seis repos.

### 4.1 Formato

```markdown
# STATUS

## Última actualización: 2026-07-15 14:30 (America/Lima)

## En progreso

### Claude Code

- **Repo:** Jin_Core
- **Rama:** `feature/claude/hitl-classifier`
- **Descripción:** Implementando el clasificador HITL para tools de Google Calendar.
- **Archivos activos:**
  - `src/hitl/classifier.ts`
  - `src/hitl/classifier.spec.ts`
- **Estado:** En desarrollo activo. No editar estos archivos.

### Antigravity

- **Repo:** Jin_Core
- **Rama:** `feature/antigravity/canvas-integration`
- **Descripción:** Scaffolding de la integración con Canvas LMS.
- **Archivos activos:**
  - `src/integrations/canvas/**`
- **Estado:** En desarrollo activo. No editar estos archivos.

## Bloqueados / esperando

- Canvas integration bloquea la Fase 3 hasta que el Executor esté listo.
- Owner debe decidir: ¿scope de Google OAuth incluye Drive o solo Gmail+Calendar?

## Recientemente completado (últimos 7 días)

- 2026-07-14: [Jin_Core] `feature/claude/audit-hash-chain` merged.
- 2026-07-13: [Jin_Infra] `feature/antigravity/k3s-manifests` merged.
```

### 4.2 Reglas

- **Actualiza al empezar** una sesión de trabajo: añade tu repo, tu rama, tus archivos.
- **Actualiza al terminar** una sesión: mueve a "recientemente completado" o marca como "en pausa".
- **Antes de editar** un archivo de Jin_Core, verifica que no aparezca en la sección "En progreso" del otro IDE. En los demás repos basta con respetar el owner primario.
- Si detectas colisión: **detente**, escribe al owner, negocia.

---

## 5. Protocolo de handoff

Cuando una tarea termina en un IDE y continúa en otro:

1. El IDE que termina hace:
   - Commit + push de todo su trabajo.
   - PR abierto o merge a `main` si está listo.
   - Si cambió un contrato: regenerar `contracts/openapi.json` y verificar que quedó commiteado.
   - Actualiza `STATUS.md` marcando su parte como completa.
   - Escribe una nota de handoff en el PR o en `STATUS.md`: "Antigravity: el endpoint `/approvals` ya está en el contrato de core; regenera tipos en web con `pnpm generate:api` y consume desde ahí."

2. El IDE que empieza:
   - Pull latest de `main` del repo que va a tocar.
   - Lee la nota de handoff.
   - Si consume contratos: `pnpm generate:api` antes de escribir código.
   - Actualiza `STATUS.md` con su nueva rama.

---

## 6. Selección de modelos por IDE

### 6.1 En Claude Code

| Situación                                       | Modelo        | Razón                                       |
| ----------------------------------------------- | ------------- | ------------------------------------------- |
| Trabajo diario, features rutinarias, bugfixes   | **Sonnet 5**  | Default. Suficiente, rápido, económico      |
| Diseño de arquitectura, decisiones no triviales | **Opus 4.8**  | Consistencia superior en razonamiento largo |
| Debug de bug sutil o revisión de seguridad      | **Opus 4.8**  | Menos probable a dejar pasar flaws          |
| Preguntas triviales                             | **Haiku 4.5** | Barato y rápido                             |

Ver `MODEL_ROUTING.md` para la tabla completa.

### 6.2 En Antigravity

| Situación                                          | Modelo                          | Razón                                                    |
| -------------------------------------------------- | ------------------------------- | -------------------------------------------------------- |
| Trabajo autónomo largo, coding agente              | **Gemini 3.5 Flash**            | Default. Mejor en Terminal-Bench y MCP Atlas que 3.1 Pro |
| Manipulación de codebase grande (>500k tokens)     | **Gemini 3.1 Pro**              | 2M context window único en el mercado                    |
| Análisis de PDFs de Canvas / docs largos           | **Gemini 3.1 Pro**              | Multimodal + contexto largo                              |
| Antigravity Agent para tareas multi-paso autónomas | **antigravity-preview-05-2026** | Sandbox managed integrado                                |

### 6.3 Cuándo cambiar de IDE

Cambia de Claude Code a Antigravity si:

- La tarea requiere leer >20 archivos a la vez.
- Necesitas manipular un PDF grande o docs.
- Vas a hacer scaffolding masivo (crear 50+ archivos).

Nota: "trabajo browser-based" dejó de ser motivo automático para preferir Antigravity — desde que `Jin_Web` es de Claude Code (2026-08-02), Claude Code prueba su propio dashboard end-to-end con sus herramientas de navegador.

Cambia de Antigravity a Claude Code si:

- La tarea es de precisión quirúrgica (refactor de 10 líneas críticas).
- Estás tocando HITL, audit, budget, security, memory, o el Executor.
- Necesitas TDD estricto con muchas iteraciones cortas.
- Estás debuggeando un bug sutil.

---

## 7. Reglas anti-colisión

### 7.1 Nunca editar simultáneamente

Nunca dos IDEs sobre el mismo archivo. Punto. Aunque estén "cerca en tiempo" (uno lo terminó hace 5 minutos), verifica que el push esté hecho y el otro IDE haya hecho pull antes de tocar. Solo aplica de facto a Jin_Core; en el resto lo previene el ownership por repo.

### 7.2 Regla del "reciente"

Si un archivo de core fue tocado en `main` en las últimas 24 h por el otro IDE, dale prioridad de review antes de modificarlo. Su LLM tenía contexto fresco de ese archivo.

### 7.3 Dependencias y config de cada repo

`package.json`, `tsconfig`, workflows: nunca simultáneamente. Añadir una dependencia = ping al owner primero. Ventaja del multi-repo: el cambio solo puede romper a ese repo.

### 7.4 Migrations

Antigravity crea nuevas migrations en core. Claude Code no. Excepción: bugfix en una migration ya creada.

### 7.5 Tests

Cada IDE escribe sus propios tests. Nunca modifiques tests que otro IDE escribió salvo que el test esté roto (en ese caso, díselo al owner).

### 7.6 Contratos

Solo el owner del repo productor cambia su API. El consumidor **regenera**, jamás edita los tipos generados a mano.

---

## 8. Cuando algo sale mal

### 8.1 Conflicto de merge

- No lo resuelve el LLM automáticamente.
- Escala al owner con contexto: qué repo, qué rama, qué archivos, qué intenta cada lado.

### 8.2 STATUS.md desactualizado

- El primer IDE en detectarlo lo corrige y avisa al owner.
- El owner decide si hay pérdida real de trabajo.

### 8.3 Un IDE editó territorio del otro

- El IDE afectado revierte los cambios en su rama.
- Escribe issue con etiqueta `coordination-fail` para revisión.
- El owner decide si la lógica se traslada al owner correcto.

### 8.4 Duda sobre ownership

- Si un archivo nuevo de core no encaja claramente en el mapa: elige el IDE cuyo trabajo lo motivó, y añádelo al mapa via PR a `WORKFLOW.md`.
- Si es un repo nuevo: el owner humano decide el owner primario al crearlo.

---

## 9. Sesiones típicas

### 9.1 Sesión mixta: nueva feature end-to-end

Ejemplo: añadir integración con Notion.

1. **Antigravity** (Gemini 3.5 Flash), en Jin_Core: scaffolding.
   - Crea `src/integrations/notion/`.
   - Instala SDK (ping al owner por la dependencia nueva).
   - Boilerplate del módulo NestJS con DTOs anotados para OpenAPI.
   - Escribe integration tests con testcontainers.
   - PR abierto, STATUS.md actualizado.

2. **Claude Code** (Sonnet 5), en Jin_Core: tools + HITL.
   - Lee el trabajo de Antigravity.
   - Declara tools en `src/tools/registry.ts` con `hitlLevel`.
   - Añade tests para el clasificador HITL cubriendo las nuevas tools.
   - Añade entradas al audit log si aplica.
   - PR abierto, STATUS.md actualizado.

3. **Owner**: revisa ambos PRs, mergea en orden. Si el contrato cambió, web/cli regeneran cuando les toque.

### 9.2 Sesión pura Claude: refactor de seguridad

Ejemplo: reforzar el sanitizer de prompt injection.

- Solo Claude Code, en Jin_Core. Rama `feature/claude/sanitizer-hardening`.
- STATUS.md declara los archivos bloqueados.
- Antigravity, si está corriendo, no toca `src/security/**`.

### 9.3 Sesión pura Antigravity: manifests de infra

Ejemplo: añadir un HorizontalPodAutoscaler para core.

- Solo Antigravity, en Jin_Infra. Rama `feature/antigravity/hpa`.
- Cero riesgo de colisión: es su repo.

---

## 10. Métricas de la colaboración

Cada mes, revisa:

- **PRs por IDE:** aproximadamente balanceado (no tiene que ser 50/50, pero un extremo indica que estás subutilizando uno).
- **Conflictos de merge en core:** debería ser cerca de 0. Si son >2/mes, el mapa de ownership interno está mal.
- **Breaking changes de contrato no anunciados:** deben ser 0. Si aparece uno, el protocolo 3.4 falló — revisa por qué.
- **Bugs por área:** si un área tiene muchos bugs, considera cambiar el IDE que la lidera.

Documenta ajustes en un ADR (`Jin_Docs/docs/adr/`) y actualiza `WORKFLOW.md`.
