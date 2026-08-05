# Recomendaciones de Claude Code — criterio propio (2026-08-05)

> **Qué es este documento y qué no es.** La auditoría de cobertura (`STATUS.md`, sección "BLUEPRINT vs. roadmap") lista funciones que el blueprint **especifica** y nadie construyó — eso es deuda contra un contrato ya escrito. Este documento es otra cosa: lo que **yo** creo que le falta a Jin en base a haber trabajado el código, y que el blueprint no pide en ningún lado. Es criterio, no contrato.
>
> **Estado: propuesto, sin aprobar.** Nada de acá se implementa hasta que el owner lo revise. Cada punto marca qué verifiqué y cómo, para que se pueda discutir sobre evidencia y no sobre opinión.

---

## P0 — Antes o junto con el despliegue (Fase 7.1)

### 1. La readiness probe, tal como está hoy, mentiría

**Verificado:** `Jin_Core/src/app.controller.ts` expone un único `@Get()` que devuelve un string estático. Su propio comentario lo dice: "Liveness básico". No consulta Postgres ni Redis.

**Por qué importa:** BLUEPRINT §12.2 declara "ReadinessProbe obligatoria; sin ella no pasa el linter de CI". Cuando la Fase 7.1 escriba los manifests, la ruta obvia a apuntar es `/` — y ahí K8s marcaría el pod como *listo para recibir tráfico* aunque Postgres esté caído. El rolling update de §12.2 (`maxSurge: 1, maxUnavailable: 0`) depende de que readiness signifique algo: si siempre dice "listo", K8s mata la réplica vieja antes de que la nueva pueda servir de verdad, y el "cero downtime" es ficticio.

**Propuesta:** endpoint `/health/ready` separado del liveness, que verifique Postgres y Redis con timeout corto, y devuelva 503 si alguno falla. Liveness sigue siendo el `/` actual (si el proceso responde, está vivo — reiniciarlo por un Postgres caído sería contraproducente). Coordinar con quien haga 7.1 para que la probe apunte al nuevo endpoint.

**Esfuerzo:** bajo. **Riesgo de no hacerlo:** el primer incidente de producción se ve como "el dashboard responde pero todo da error".

### 2. El historial de chat crece sin cota

**Verificado:** cero poda en los tres lados — ni `AgentService.runTurn` (que hace `[...input.history, nuevo mensaje]`), ni `Jin_Web/app/routes/chat.tsx`, ni `Jin_CLI/source/components/ChatView.tsx`. El `/chat` (REST y WS) es stateless por diseño: el cliente acumula `ModelMessage[]` y reenvía **todo** en cada turno.

**Por qué importa:** el costo por turno crece linealmente con la longitud de la sesión (turno N paga por N-1 turnos de contexto), así que el costo acumulado de una sesión crece de forma cuadrática. El budget guard (Fase 4.1) *ve* el gasto y puede cortar, pero no puede prevenir la causa estructural — cortaría en medio de una conversación por un problema que no es del usuario. Y con sesión suficientemente larga se llega al límite de contexto del modelo y el turno falla duro.

Esto no es hipotético para el uso declarado (§1.3: "~10-50 tareas ligeras al día"): una sesión de chat larga en un día de trabajo llega ahí.

**Propuesta:** decidir una política explícita y aplicarla en `Jin_Core` (no en cada cliente, o se implementa distinto tres veces): ventana deslizante por número de turnos, o por presupuesto de tokens, con resumen del tramo podado. La memoria extendida (`src/memory/`, Fase 4.3) ya existe para justamente esto — el tramo viejo se consolida ahí en vez de perderse.

**Esfuerzo:** medio (la decisión de política es lo caro, no el código). **Riesgo de no hacerlo:** gasto silencioso y un fallo duro difícil de explicar.

---

## P1 — Fricción real, esfuerzo bajo

Los tres salieron de trabajar el repo hoy: cada uno me costó tiempo de diagnóstico y se lo va a costar al siguiente.

### 3. Los scripts standalone no cargan `.env`

**Verificado:** `src/db/migrate.ts` llama `validateEnv(process.env)` sin ningún `dotenv` previo. Lo mismo aplica a `generate-contract.ts` y `hash-password.ts`. Solo la app NestJS carga `.env` (vía `ConfigModule.forRoot()`).

**Síntoma real:** `pnpm run db:migrate` falla con las 16 variables en `undefined` aunque `.env` exista y esté correcto. El workaround (exportar al shell) es frágil: el hash argon2 contiene `$` y comas, y según el estado del shell se corrompe silenciosamente — me pasó hoy, y el error resultante no apunta a la causa.

**Propuesta:** `import 'dotenv/config'` al tope de los tres entrypoints (dotenv ya viene transitivamente con `@nestjs/config`, verificar antes de asumirlo) o `dotenv-cli` en los scripts de `package.json`. Cualquiera de las dos, pero que sea una y esté documentada.

### 4. El config de e2e colisiona con el Redis de desarrollo

**Verificado:** `test/vitest-e2e.config.ts` declara en su propio comentario que usa valores "sin tocar Postgres/Redis reales", pero apunta a `redis://localhost:6379` — exactamente el puerto que publica `Jin_Infra/docker-compose.dev.yaml`.

**Síntoma real:** con el compose levantado (que es lo que recomienda `CLAUDE.md` §4 para desarrollo), los 19 tests e2e fallan con `Connection is closed`, incluido uno tan trivial como `GET /`. Hoy me hizo perder tiempo persiguiendo una regresión inexistente: pasaron los 19 en cuanto detuve el contenedor.

**Propuesta:** apuntar el config e2e a un puerto donde nunca haya nada (ej. `6399`), que es lo que la intención declarada del comentario ya asume (`lazyConnect: true`, nunca conecta de verdad).

### 5. El tooling ya no entra en el heap por defecto de Node

**Verificado:** `pnpm run start:dev`, `pnpm run generate:contract` y `pnpm run lint` mueren con `FATAL ERROR: Reached heap limit` en esta máquina (8 GB), y funcionan con `NODE_OPTIONS=--max-old-space-size=4096`.

**Por qué importa:** hoy es conocimiento tribal. Quien clone el repo se topa con un crash de V8 sin relación aparente con lo que estaba haciendo. Y va a empeorar: el proyecto solo crece.

**Propuesta:** fijar el flag en los propios scripts de `package.json` (afecta solo al tooling, no al runtime de producción, donde los límites los pone el manifest de K8s).

---

## P2 — Calidad sostenida

### 6. Jin_Web no tiene tests de navegador en CI

**Verificado:** cero Playwright en `Jin_Web/package.json`, pese a que `CLAUDE.md` §4 lista `pnpm test:e2e` como comando esperado del repo desde la Fase 6.

**Por qué importa, con evidencia de esta misma sesión:** los dos bugs reales del dashboard fueron **invisibles** a `typecheck`, `lint` y `build`, y aparecieron solo al abrirlo en un navegador de verdad — la pantalla en negro de `/hitl` (`openapi-fetch` resolviendo `{data: undefined, error: undefined}`) y los skeletons infinitos por falta de manejo de `isError`. Yo los corrí a mano; sin eso en CI, el próximo de esa familia se descubre en producción.

**Propuesta:** un smoke de Playwright en CI que haga login y recorra las rutas principales verificando que ninguna quede en blanco ni tire error de consola. No hace falta cobertura exhaustiva — el 80% del valor está en "la página carga y no explota".

### 7. No hay procedimiento para revertir una acción del agente

**Verificado por ausencia:** existe kill switch (runaway) y budget guard (costo), y el audit log permite reconstruir qué pasó. No existe nada para deshacer.

**Por qué importa:** todo el sistema HITL está diseñado para el caso "¿autorizo esto?", y es sólido. Pero no cubre "lo autoricé y estuvo mal", que con un agente autónomo es cuestión de tiempo. Es especialmente relevante para el nivel `notify`, que ejecuta **sin** aprobación previa (crear evento en Calendar, `git commit`): ahí el owner se entera después, y hoy no tiene herramienta para revertir más allá de hacerlo a mano en cada servicio.

**Propuesta:** no código todavía — primero decidir el alcance. Mi sugerencia es empezar por un runbook (qué se puede revertir y cómo, por tool) y recién después evaluar si vale una tool de compensación. Documentar que ciertas acciones son irreversibles por naturaleza es un resultado válido y útil.

---

## P3 — Higiene de documentación

### 8. El diagrama de arquitectura del blueprint todavía muestra BullMQ

**Verificado:** `BLUEPRINT.md` §2 dibuja "jin-core (NestJS) · BullMQ+Redis", y §3.4 lo declara como "todas las colas de tareas asíncronas". BullMQ está descartado (cero en `package.json`) y así consta en `STATUS.md`/`PROMPTS.md`.

**Por qué importa:** el blueprint se declara a sí mismo "la fuente de verdad" (§1). Quien lo lea de cero — un agente nuevo en una sesión futura, sin este contexto — va a creerle al diagrama. Ya pasó algo así con la versión de React Router en §8.1, corregido en la Fase 6.2.

**Propuesta:** actualizar §2 y §3.4 con una nota de la desviación, mismo criterio que se usó para §8.1.1 (PWA) y §8.1 (React Router).

### 9. Iconos PWA reales

Ya documentado como gap conocido en `STATUS.md` (Fase 6.2+6.3): hoy es el `favicon.svg` del template, no el ícono 512px maskable que pide el diseño §09. Requiere un asset de diseño, no código.

---

## Resumen para decidir

| # | Recomendación | Prioridad | Esfuerzo | Bloquea despliegue |
| --- | --- | --- | --- | --- |
| 1 | Readiness probe real | P0 | Bajo | Sí (hace ficticio el cero-downtime) |
| 2 | Poda del historial de chat | P0 | Medio | No, pero el costo corre desde el día 1 |
| 3 | `.env` en scripts standalone | P1 | Muy bajo | No |
| 4 | Colisión e2e / Redis local | P1 | Muy bajo | No |
| 5 | Heap de Node en tooling | P1 | Muy bajo | No |
| 6 | Playwright en CI de Jin_Web | P2 | Bajo | No |
| 7 | Reversión de acciones del agente | P2 | Alto (decisión) | No |
| 8 | Diagrama §2 desactualizado | P3 | Muy bajo | No |
| 9 | Iconos PWA reales | P3 | — (diseño) | No |

**Mi recomendación de secuencia:** 3, 4 y 5 juntos en un PR único de fricción (media hora, cero riesgo). 1 antes o junto con 7.1. 6 cuando haya un momento tranquilo — su valor es preventivo y ya está demostrado. 2 y 7 merecen una conversación antes de código: ambos requieren una decisión de política del owner, no una implementación obvia.
