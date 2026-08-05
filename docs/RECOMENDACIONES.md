# Recomendaciones de Claude Code — criterio propio

> **Qué es este documento y qué no es.** La auditoría de cobertura (`STATUS.md`, sección "BLUEPRINT vs. roadmap") lista funciones que el blueprint **especifica** y nadie construyó — eso es deuda contra un contrato ya escrito. Este documento es otra cosa: lo que **yo** creo que le falta a Jin en base a haber trabajado el código, y que el blueprint no pide en ningún lado. Es criterio, no contrato.
>
> **Estado: propuesto, sin aprobar** (salvo lo marcado ✅). Nada de acá se implementa hasta que el owner lo revise. Cada punto marca qué verifiqué y cómo, para que se pueda discutir sobre evidencia y no sobre opinión.
>
> **Ronda 1 (2026-08-05)** — puntos 1-9, salidos de trabajar `Jin_Core`/`Jin_Web`. Los dos P0 ya están cerrados.
> **Ronda 2 (2026-08-05)** — puntos 10-24, salidos de una auditoría cross-repo pedida por el owner sobre los 6 repos (`Jin_Core`, `Jin_Web`, `Jin_CLI`, `Jin_Executor`, `Jin_Infra`, `Jin_Docs`). Ver más abajo.

---

## Ronda 1

## P0 — Antes o junto con el despliegue (Fase 7.1)

### 1. ✅ RESUELTO — La readiness probe, tal como está hoy, mentiría

**Verificado:** `Jin_Core/src/app.controller.ts` expone un único `@Get()` que devuelve un string estático. Su propio comentario lo dice: "Liveness básico". No consulta Postgres ni Redis.

**Por qué importa:** BLUEPRINT §12.2 declara "ReadinessProbe obligatoria; sin ella no pasa el linter de CI". Cuando la Fase 7.1 escriba los manifests, la ruta obvia a apuntar es `/` — y ahí K8s marcaría el pod como *listo para recibir tráfico* aunque Postgres esté caído. El rolling update de §12.2 (`maxSurge: 1, maxUnavailable: 0`) depende de que readiness signifique algo: si siempre dice "listo", K8s mata la réplica vieja antes de que la nueva pueda servir de verdad, y el "cero downtime" es ficticio.

**Propuesta:** endpoint `/health/ready` separado del liveness, que verifique Postgres y Redis con timeout corto, y devuelva 503 si alguno falla. Liveness sigue siendo el `/` actual (si el proceso responde, está vivo — reiniciarlo por un Postgres caído sería contraproducente). Coordinar con quien haga 7.1 para que la probe apunte al nuevo endpoint.

**Esfuerzo:** bajo. **Riesgo de no hacerlo:** el primer incidente de producción se ve como "el dashboard responde pero todo da error".

**Resuelto en [Jin_Core PR #22](https://github.com/JFrnck/Jin_Core/pull/22)** (2026-08-04): `/health/live` + `/health/ready` (Postgres duro → 503; Redis solo degrada → 200 `degraded`). Handoff vivo hacia Fase 7.1 — ver punto 24.

### 2. ✅ RESUELTO — El historial de chat crece sin cota

**Verificado:** cero poda en los tres lados — ni `AgentService.runTurn` (que hace `[...input.history, nuevo mensaje]`), ni `Jin_Web/app/routes/chat.tsx`, ni `Jin_CLI/source/components/ChatView.tsx`. El `/chat` (REST y WS) es stateless por diseño: el cliente acumula `ModelMessage[]` y reenvía **todo** en cada turno.

**Por qué importa:** el costo por turno crece linealmente con la longitud de la sesión (turno N paga por N-1 turnos de contexto), así que el costo acumulado de una sesión crece de forma cuadrática. El budget guard (Fase 4.1) *ve* el gasto y puede cortar, pero no puede prevenir la causa estructural — cortaría en medio de una conversación por un problema que no es del usuario. Y con sesión suficientemente larga se llega al límite de contexto del modelo y el turno falla duro.

Esto no es hipotético para el uso declarado (§1.3: "~10-50 tareas ligeras al día"): una sesión de chat larga en un día de trabajo llega ahí.

**Propuesta:** decidir una política explícita y aplicarla en `Jin_Core` (no en cada cliente, o se implementa distinto tres veces): ventana deslizante por número de turnos, o por presupuesto de tokens, con resumen del tramo podado. La memoria extendida (`src/memory/`, Fase 4.3) ya existe para justamente esto — el tramo viejo se consolida ahí en vez de perderse.

**Esfuerzo:** medio (la decisión de política es lo caro, no el código). **Riesgo de no hacerlo:** gasto silencioso y un fallo duro difícil de explicar.

**Resuelto en [Jin_Core PR #23](https://github.com/JFrnck/Jin_Core/pull/23)** (2026-08-04): criterio híbrido (techo de tokens + últimos K turnos verbatim), tramo viejo resumido y consolidado a `src/memory/`, devuelto al caller como `compactedHistory`. Detalle completo y hallazgos de seguridad en `STATUS.md`. **Handoff vivo:** `Jin_CLI` debe adoptar `compactedHistory` o el fix queda sin consumidor real.

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

---

## Ronda 2 — auditoría cross-repo (2026-08-05)

> Pedida por el owner tras cerrar los dos P0 de la Ronda 1: "revisá el proyecto y buscá qué más falta que no esté documentado". Cubre los 6 repos. Todo lo de acá está **verificado contra el código**, con archivo:línea — no son sospechas.
>
> **Nota de corrección:** en un reporte verbal previo dije que `Jin_Infra` no tenía CI. Es falso — sí tiene (`.github/workflows/ci.yaml`, con `kustomize build` + `kube-linter` + `shellcheck` + tests `bats`), y es de los más completos del proyecto. El repo sin CI es `Jin_Docs`.

### P0 — Seguridad e integridad, antes del deploy real

#### 10. ✅ RESUELTO (parcial, ver nota) — Zip-slip en `startPreviewService`: un `files` malicioso escribe fuera de `/workspace`

**Verificado:** `Jin_Executor/src/preview-service/preview-service-request.schema.ts:9` valida `files: z.record(z.string().min(1), z.string())` — la clave es el path de destino y **no se restringe su contenido**. `tar-payload.ts:71-85` escribe esa clave tal cual en el header USTAR sin sanear `../` ni rutas absolutas, y el init container la extrae con `tar xz -C /workspace` (`service-pod-spec.builder.ts:88`). Una clave `"../../app/evil.js"` escapa del directorio de trabajo.

**Se agrava con un segundo hallazgo:** el container `app` de `buildServicePodSpec` (`service-pod-spec.builder.ts:106-129`) **no declara `readOnlyRootFilesystem`**, a diferencia del pod de `runCode` (`pod-spec.builder.ts:83`, que sí lo pone). O sea: hay a dónde escribir.

**Por qué el HITL no alcanza como defensa:** `startPreviewService` es `hitlLevel: 'confirm'`, pero el owner aprueba un `planSummary` ("levantar preview de X"), no audita cada clave del mapa `files`. Un path traversal ahí es invisible en la pantalla de aprobación.

**Propuesta:** rechazar en el schema Zod toda clave que sea absoluta o contenga `..` (fail-fast, antes de construir el tar), **y** poner `readOnlyRootFilesystem: true` + un `emptyDir` escribible solo donde haga falta. Los dos, no uno: defensa en profundidad. Cubrirlo con un test explícito — `tar-payload.spec.ts` y `service-pod-spec.builder.spec.ts` hoy no prueban ningún path con `..`.

**Esfuerzo:** bajo. **Riesgo de no hacerlo:** el único camino del sistema donde un LLM controla nombres de archivo escritos a disco no tiene barrera.

**Resuelto en [Jin_Executor PR #6](https://github.com/JFrnck/Jin_Executor/pull/6)** (2026-08-05): `isSafeRelativePath()` en dos capas (`tar-payload.ts` + Zod en el borde HTTP) — esto es lo que cierra el zip-slip en sí, y está confirmado por CI real con K3s vivo. **`readOnlyRootFilesystem: true` en el container `app` se intentó como defensa en profundidad adicional y se revirtió en el mismo PR:** el CI real (no una sospecha) mostró el smoke test "camino feliz" colgado 180s, con cuatro minutos de silencio total en el log — ni siquiera el timeout propio de `waitUntilPodRunning` (60s) llegó a dispararse, lo que apunta a algo más temprano que el pod nunca sirviendo HTTP, distinto de la flakiness de red ya documentada en ADR 0003 (esa es rápida, con reintentos que sí loguean). Sin acceso a un clúster K3s real para diagnosticar la causa exacta, no se forzó el cambio sin poder verificarlo — queda documentado en el código como intentado y revertido, no descartado en silencio, pendiente de retomar con logging más granular en el test o acceso a un clúster real. **La corrección primaria (el zip-slip) no depende de esto** y queda cerrada. **Hallazgo lateral corregido de paso:** `Jin_Executor/contracts/openapi.json` estaba desactualizado desde el PR #5 (Fase 5.5) — faltaban los 3 endpoints `/services*` por completo. Sin impacto en consumidores (el `executor-client` de Jin_Core está escrito a mano, no generado desde ese contrato) — era deuda de documentación pura.

#### 11. ✅ RESUELTO — Tres notificaciones críticas quedaron esperando una fase que ya se completó

**Verificado:** tres callers difieren su notificación a "Fase 2.4 (bot Telegram)" — una fase **cerrada desde el PR #5**, hace semanas. Ninguno se recableó:

- `Jin_Core/src/hitl/timeout.service.ts:67-70` — escalado a las 12h sin respuesta: `logger.warn` + el comentario literal *"Notificación real llega en Fase 2.4 (bot Telegram)"*.
- `Jin_Core/src/hitl/timeout.service.ts:81-83` — aprobación **abandonada** tras 24h: solo `logger.error`.
- `Jin_Core/src/audit/chain-verification.service.ts:40-43` — **audit chain corrupta**: `logger.error` + *"Alerta Telegram real: Fase 2.4 (bot grammY). Por ahora solo el log."*

**Por qué importa:** son, respectivamente, "tu aprobación está por vencer", "tu aprobación venció y la acción se descartó" y "el registro de auditoría fue manipulado". Los tres son eventos donde el sistema necesita al humano, y los tres terminan en un log que nadie mira. El owner pide algo por Telegram, no contesta, y a las 24h la acción se descarta **en silencio** — desde su lado es indistinguible de que Jin nunca hizo nada.

**La capacidad ya existe y está probada:** `TelegramBotService.checkBudgetAlerts()` (`telegram-bot.service.ts:605-680`) hace `bot.api.sendMessage` proactivo para las alertas de presupuesto (Fase 4.1 sí lo cableó). No hay que construir nada nuevo — solo conectar los tres callers viejos al canal que ya funciona.

**Esfuerzo:** bajo. **Riesgo de no hacerlo:** el HITL parece completo y no lo está; el modo de falla es silencioso, que es el peor para un sistema de aprobaciones.

**Resuelto en [Jin_Core PR #24](https://github.com/JFrnck/Jin_Core/pull/24)** (2026-08-05): `TimeoutService` emite eventos (`HITL_APPROVAL_ESCALATED_EVENT`/`HITL_APPROVAL_ABANDONED_EVENT`) que `TelegramBotService` escucha con `@OnEvent`; la corrupción del audit log se resuelve con polling (`checkAuditIntegrityAlert`, mismo criterio que el kill switch) — evita el import circular que una inyección directa hubiera creado.

#### 12. ✅ RESUELTO — El bloqueo del audit log por corrupción no sobrevive a un redeploy

**Verificado:** `ChainVerificationService.locked` (`chain-verification.service.ts:17`) es un booleano **en memoria**. Si el barrido diario detecta corrupción, bloquea escrituras (`audit.service.ts:151` lo consulta) — pero un restart del pod lo devuelve a `false` y las escrituras se reanudan sobre una cadena corrupta, sin que nadie intervenga.

**Lo llamativo:** el propio repo ya identificó este gap y lo dejó anotado **solo en comentarios de código**, en dos lugares (`budget/kill-switch.service.ts:29` y `db/schema.ts:135`), ambos diciendo *"a diferencia de `ChainVerificationService.locked` (en memoria, gap preexistente ajeno a este módulo)"*. Nunca salió de ahí a `STATUS.md` ni a este documento — por eso lo formalizo acá.

**Propuesta:** persistir el lock igual que el kill switch (fila singleton en Postgres, patrón ya escrito en `budget_kill_switch`), con desbloqueo manual explícito. El precedente y la justificación ya están escritos: *"un simple redeploy no debe despausar el sistema sin intervención humana"* — aplica idéntico acá.

**Esfuerzo:** bajo (el patrón ya existe, se copia). **Riesgo de no hacerlo:** la garantía "un audit log corrupto es peor que uno detenido" (regla de oro del propio módulo) se rompe con un `kubectl rollout restart`.

**Resuelto en [Jin_Core PR #24](https://github.com/JFrnck/Jin_Core/pull/24)** (2026-08-05): tabla singleton `audit_chain_lock` (migración 0007), mismo patrón que `budget_kill_switch`. **Bug crítico encontrado en el camino:** `AuditService.appendRow()` chequeaba `isLocked()` sin `await` — con el método vuelto async eso habría sido siempre truthy (un `Promise` nunca es falsy), bloqueando TODAS las escrituras del audit log permanentemente. Corregido con el `await` explícito.

#### 13. ✅ RESUELTO — El audit log permanente pierde `actor` y `external_inputs_summary` al resolverse

**Verificado:** `AuditService.recordApproval` (`audit.service.ts:69-85`) y `recordRejection` (`:87-104`) hardcodean `externalInputsSummary: null` y **ni siquiera reciben `actor`** en su input. Esos dos datos sí viven en `pending_approvals` (columnas agregadas en el PR #21), pero esa fila **se borra al resolver** (`dual-confirm.service.ts:168`, documentado en `schema.ts:47`).

**Por qué importa:** `AGENTS.md` §5.1 punto 3 exige poder reconstruir qué contenido externo influyó en una decisión HITL. Hoy eso se puede ver *antes* de aprobar (dashboard, PR #21) pero se pierde para siempre *después* — justo en el registro permanente, que es el que importa para una investigación post-incidente. La fila terminal de `audit_log` dice "aprobado por owner" sin decir quién lo pidió ni qué lo influyó.

**Propuesta:** copiar ambos campos desde `pending_approvals` a la fila terminal de `audit_log` antes de borrar el pendiente. Es un cambio acotado, sin migración nueva (las columnas ya existen en `audit_log`).

**Esfuerzo:** bajo. **Riesgo de no hacerlo:** el rastro de auditoría es incompleto exactamente en el caso para el que existe.

**Resuelto en [Jin_Core PR #24](https://github.com/JFrnck/Jin_Core/pull/24)** (2026-08-05), con un diseño distinto al propuesto arriba: copiar `actor`/`externalInputsSummary` a `recordApproval`/`recordRejection` hubiera sobrecargado el campo `actor` existente (ahí significa "quién aprobó" = siempre el owner, no "quién pidió la tool"). El fix real va en `DualConfirmService.createPendingApproval()` — escribe una fila `tool_call`/`pending` permanente en `audit_log` al crear el pending (mismo patrón que ya usan las tools auto/notify), correlacionada por `requestId` con la fila terminal. Cubre los 5 callers reales sin tocar ninguno.

#### 14. La contraseña de `jin login` queda en el historial del shell

**Verificado:** `Jin_CLI/source/app.tsx:20` toma la contraseña de `args[0]`; `cli.tsx:12,21` documentan `jin login <contraseña>`.

**Por qué importa:** un argumento de shell queda en `~/.zsh_history` en texto plano y es visible por `ps aux` / `/proc` para cualquier otro proceso o usuario de la máquina mientras corre. Es la contraseña maestra del sistema (única credencial del owner, Argon2id contra `OWNER_PASSWORD_HASH`).

**Propuesta:** prompt interactivo enmascarado (Ink ya soporta input oculto) como camino por defecto, aceptando el argumento posicional solo si se pasa explícitamente algo tipo `--password-stdin`. Nota: el almacenamiento posterior del token en `~/.config/jin/auth.json` con `0600` **ya está bien resuelto** — el problema es solo la entrada.

**Esfuerzo:** muy bajo.

#### 15. Dos de los tres backups nunca se verifican restaurables

**Verificado:** `Jin_Infra/scripts/backup/verify-restore.sh:46` descarga, descifra y restaura únicamente el prefijo `postgres`. Existen además `dump-redis.sh` y `dump-memory.sh` (la memoria sqlite-vec del agente, Fase 4.3) — **ninguno de los dos se restaura ni se verifica nunca**.

**Por qué importa:** el comentario en la línea 3 del propio script dice *"ningún backup no probado cuenta como backup"*. Hoy esa frase es literalmente cierta para 2 de 3 tipos de datos. `memory.db` es especialmente sensible: es el único store que **no** está en Postgres, así que no lo cubre ninguna otra red de seguridad.

**Propuesta:** extender `verify-restore.sh` a los tres prefijos. Para `memory.db` la verificación puede ser tan simple como abrir el sqlite restaurado y contar filas de la tabla `vec0` — pero tiene que correr.

**Esfuerzo:** bajo. **Riesgo de no hacerlo:** se descubre el día que haga falta restaurar.

### P1 — Robustez y correctitud

#### 16. Nadie escucha `disconnect` del WebSocket: la UI queda colgada para siempre

**Verificado, en los dos clientes:**
- `Jin_Web/app/features/chat/useChat.ts:30-63` — no registra handler de `disconnect`. Si el socket cae con `pending = true` esperando `chat:response`, el turno queda en "trabajando…" indefinidamente: sin timeout, sin error, sin reintento, y **sin poder mandar otro objetivo** porque `pending` nunca vuelve a `false`.
- `Jin_CLI/source/api/ws-chat.ts:28` — `reconnection: false` explícito, ningún reintento automático. `ChatView.tsx:28-54` no pasa `onDisconnect`, así que el indicador "🟢 WS" queda mintiendo tras una caída (el fallback a REST sí funciona, porque chequea `socketRef.current.connected` real).

**Propuesta:** manejar `disconnect` en ambos: liberar el estado `pending` con un mensaje de error honesto ("se perdió la conexión, reintentá"), y en la CLI habilitar la reconexión de socket.io. **No** reenviar el turno automáticamente al reconectar — un turno de agente puede haber ejecutado tools con efectos reales; reintentarlo a ciegas es peor que pedirle al usuario que decida.

#### 17. Mutaciones sin `catch` en el dashboard: una aprobación fallida se ve igual que una exitosa

**Verificado:** cuatro sitios usan mutaciones imperativas fuera de TanStack Query sin manejar el error —
`Jin_Web/app/features/hitl/ApprovalCard.tsx:68-84` (`handleApprove`/`handleReject`, `try/finally` sin `catch`), `app/routes/preview.tsx:21-26` (`stop()` ignora `error` e invalida la query igual), `app/routes/budget.tsx:42-50` (`confirmUnpause()`), `app/routes/editor.tsx:58-83` (`requestComments()`).

**El grave es `ApprovalCard`:** si un dual-confirm expira en el servidor justo al confirmar (o el 409 de "segunda confirmación demasiado pronto" salta), la UI no muestra nada y el operador queda creyendo que **aprobó una acción irreversible que en realidad no se ejecutó**. Las rutas que usan `useQuery` directamente sí manejan `isError` bien — el gap está acotado a estas mutaciones.

**Propuesta:** pasarlas por `useMutation` con `onError`, o al menos un `catch` que muestre el error. `unwrap()` (`api-client.ts`) ya lanza; solo falta que alguien lo agarre.

#### 18. `js-yaml` de producción con un advisory alto (Jin_Core)

**Verificado:** `package.json:49` declara `"js-yaml": "^5.2.1"` como dependencia de **producción**; la instalada es exactamente `5.2.1`, y el advisory (DoS por parsing exponencial en flow collections) cubre `>=5.0.0 <=5.2.1`. Se usa al arrancar, en `models-config.schema.ts:2`, `agent-config.schema.ts:2` y `budget-config.schema.ts:2`.

**Honestidad sobre el riesgo real:** los tres YAML que parsea son archivos del propio repo, no entrada de un atacante — la explotabilidad práctica hoy es baja. Pero es una dependencia de producción con advisory alto y el bump es trivial, así que no hay razón para dejarlo. `pnpm audit` completo en `Jin_Core` da 10 high / 6 moderate; el resto es todo `devDependencies` (esbuild, brace-expansion, postcss vía tooling) que no llega al bundle.

#### 19. `monaco-editor` arrastra `dompurify` vulnerable (Jin_Web, producción)

**Verificado:** `monaco-editor@0.56.0` es dependencia de producción y trae `dompurify` con varios GHSA abiertos. Superficie real: el editor renderiza comentarios generados por el LLM (Fase 6.3).

**Propuesta:** revisar si hay versión de monaco con el `dompurify` parcheado; si no, dejarlo anotado con fecha y revisar en el siguiente ciclo, no ignorarlo en silencio.

#### 20. CI que no puede fallar: `Jin_CLI` y `Jin_Web`

**Verificado, comparando los 6 workflows:**
- `Jin_CLI/.github/workflows/ci.yml:21` — `pnpm run test || true`. **Un test roto nunca rompe el CI.** Además no tiene paso de `lint`, que Core, Executor y Web sí tienen. Esto es peor que no tener el paso: da una señal verde falsa.
- `Jin_Web/.github/workflows/ci.yml` — no ejecuta `test` en absoluto (sí lint, typecheck y build).
- `Jin_Docs` — sin ningún workflow.
- `Jin_Core`, `Jin_Executor`, `Jin_Infra` — completos, sin `|| true` en pasos de verificación.

**Propuesta:** sacar el `|| true` de CLI (y arreglar lo que rompa, si rompe algo) y agregarle `lint`. En Web, el paso de test entra junto con el punto 6 (Playwright) o con 21. `Jin_Docs` es el menos urgente — a lo sumo un linter de links rotos.

#### 20.b El `|| true` ya causó un daño real: los tipos del CLI están desincronizados

**Verificado** (regenerando los tipos desde el contrato de `main` de `Jin_Core` a un archivo temporal y diffeando contra los committeados, ignorando diferencias de formato):

- **`Jin_Web/app/lib/api-types.ts`: en sincronía.** Sin drift.
- **`Jin_CLI/source/api-types.ts`: drift real.** Le faltan exactamente dos campos del schema de aprobaciones pendientes:
  ```
  < actor: string | null;
  < externalInputsSummary: string | null;
  ```
  (el resto del diff es solo reformateo de XO/Prettier, sin diferencia semántica).

**Qué significa:** son los dos campos que agregó [Jin_Core PR #21](https://github.com/JFrnck/Jin_Core/pull/21) el 2026-08-04 para cumplir `AGENTS.md` §5.1 punto 3 — "solicitado por \<agente\>" e "influido por \<fuentes externas\>", visibles **antes** de que el owner apruebe. `Jin_CLI` nunca corrió `generate:api` después de ese merge, y su CI no podía avisar porque el paso termina en `|| true`.

**Consecuencia concreta, verificada en el código:** `Jin_CLI/source/components/TasksView.tsx:6-10` declara su propia interfaz local con solo `requestId`/`toolName`/`planSummary`, y el render (`:105-127`) no muestra ninguno de los dos campos nuevos. **Quien aprueba desde la terminal ve estrictamente menos información que quien aprueba desde el dashboard** — le falta justamente la traza de qué fuente externa pudo influir en la acción que está por autorizar. No es un problema de tipos: es una diferencia de superficie de seguridad entre dos clientes del mismo sistema HITL.

**Propuesta:** regenerar los tipos en `Jin_CLI` y mostrar ambos campos en `TasksView` (paridad con `ApprovalCard.tsx` del dashboard). Es trabajo en el repo de Antigravity → handoff, no lo tomo por mi cuenta. Refuerza el punto 20: el `|| true` no es hipotético, ya dejó pasar esto.

### P2 — Cobertura de tests

#### 21. Piezas de seguridad de `Jin_Core` sin ningún spec

**Verificado** (archivos sin `.spec.ts` co-ubicado, filtrando tipos/errores/módulos):
- `src/audit/chain-verification.service.ts` — el servicio que detecta manipulación del audit log y bloquea escrituras. La lógica pura (`hash-chain.ts`) **sí** está cubierta; el servicio que decide bloquear, no.
- `src/auth/ws-token.ts` — parseo de cookies y extracción de token para el handshake WebSocket. Es código de autenticación escrito a mano (el handshake WS no pasa por el pipeline de Nest), con parseo de strings — exactamente donde conviene tener tests.
- `src/chat/chat.gateway.ts` y `src/realtime/realtime.gateway.ts` — validan el JWT en `handleConnection`.
- `src/config/env.schema.ts` — la validación fail-fast de las 16+ variables de entorno.

#### 22. `Jin_Web` no tiene un solo test, de ningún tipo

**Verificado:** cero archivos `.spec.ts`/`.test.ts` en todo `app/`. El punto 6 de la Ronda 1 pedía Playwright en CI; esto es aparte y anterior — **tampoco hay tests unitarios** de lógica que no necesita navegador: `app/lib/api-client.ts` (`unwrap()`, donde ya hubo un bug real), `app/lib/use-connection-status.ts` (máquina de 4 estados), `useChat.ts`, `useApprovals.ts`.

#### 23. `Jin_CLI`: cobertura mínima

**Verificado:** un único `test.tsx` que cubre `App` (help), `LoginView` (solo el camino de error sin contraseña), `ApproveRejectView` (solo id faltante) y `auth-storage` (happy path). Sin tests para `ws-chat.ts`, la lógica WS/REST de `ChatView.tsx`, `StatusView`, `TasksView`, `MemoryView` ni `client.ts`.

### P3 — Infra y handoffs

#### 24. Handoff vivo: cablear las probes nuevas cuando exista el Deployment

**Verificado:** en `Jin_Infra` todavía **no existe** ningún Deployment/Service para las apps (`k8s/base/kustomization.yaml:5-15` no los incluye; solo hay RBAC del executor y el PVC `jin-core-data` con el comentario explícito *"antes de que exista el Deployment de core"*). O sea: hoy no hay ninguna probe apuntando a `/` que corregir — pero cuando la Fase 7.1 cree esos manifests, hay que cablear `readinessProbe` → `/health/ready` y `livenessProbe` → `/health/live`. **Nada en el repo lo garantiza automáticamente**, por eso queda escrito acá y en `STATUS.md`.

#### 25. `curl | sh` sin verificar checksum en el bootstrap

**Verificado:** `Jin_Infra/scripts/bootstrap/01-install-k3s.sh:11` instala K3s con `curl -sfL https://get.k3s.io | sh -s -` (versión pinneada vía `INSTALL_K3S_VERSION`, pero sin verificar el instalador); mismo patrón en `05-flux-bootstrap.sh:13`. Ambos ejecutan con privilegios. (Lo que **no** encontré: ningún script que asuma ser root sin verificarlo — `01` usa `sudo` explícito donde corresponde.)

**Propuesta:** pinnear el instalador a un hash conocido o usar el binario firmado del release. Es una VM que se bootstrapea una vez cada mucho tiempo, así que la prioridad es baja — pero es el momento de mayor privilegio de todo el ciclo de vida del sistema.

#### 26. ✅ RESUELTO — Init container de preview-service sin límites de recursos propios

**Verificado:** `Jin_Executor/src/preview-service/service-pod-spec.builder.ts:81-105` — el container `extract-workspace` no define `resources`, a diferencia del container `app` (`:130-136`) y del pod de `runCode` (`pod-spec.builder.ts:89-92`). Depende enteramente de que el `LimitRange` de `agents-sandbox` (que vive en `Jin_Infra`, otro repo) inyecte defaults y cubra init containers. El tamaño de `files` no está acotado por schema (documentado en `preview-service-request.schema.ts:6-8`).

**Propuesta:** declarar `resources` explícitos en el init container, sin depender de un default que vive en otro repo.

**Resuelto en [Jin_Executor PR #6](https://github.com/JFrnck/Jin_Executor/pull/6)** (2026-08-05), junto con el punto 10 — mismo archivo, mismo PR.

#### 27. (nuevo, encontrado al cerrar 10/26) `Jin_Executor/contracts/openapi.json` llevaba desactualizado desde Fase 5.5

**Verificado:** regenerar el contrato en el PR #6 agregó 58 líneas — los 3 endpoints `POST /services`, `GET /services`, `DELETE /services/{id}` faltaban por completo. `git log --oneline -- contracts/openapi.json` confirma que el archivo no se tocó desde el PR #3/#4 (Fase 5.2/rename), es decir desde **antes** de que existiera el controller de preview services (PR #5).

**Por qué importa poco hoy, pero conviene cerrar:** `Jin_Core/src/executor-client/` (el único consumidor real de esta API) está escrito a mano, no generado desde este contrato — así que no hay ningún bug de tipos en producción por esto, a diferencia del punto 20.b. Es deuda de documentación pura: `/docs` (Swagger UI del Executor) miente sobre su propia superficie, y si algún día alguien generara un cliente desde este contrato (mismo patrón que Web/CLI con Core), heredaría el hueco.

**Propuesta:** ya regenerado en el PR #6. Vale la pena que el CI de `Jin_Executor` corra `generate:contract` y falle si hay diff sin commitear (mismo problema de fondo que el punto 20: nada impide que un contrato se desincronice en silencio).

#### 28. (nuevo, abierto) `readOnlyRootFilesystem` en el container `app` de preview-service — intentado y revertido, sigue pendiente

**Contexto:** al cerrar el punto 10, se intentó `readOnlyRootFilesystem: true` en el container `app` como defensa en profundidad (el zip-slip ya cerrado en la capa de datos deja de importar tanto si además no hay dónde escribir). El CI real de `Jin_Executor` (K3s vía testcontainers) mostró el smoke test "camino feliz" de `preview-service.service.integration.spec.ts` colgado el timeout completo de 180s, con **cuatro minutos de silencio total** en el log — ni siquiera el timeout propio de `waitUntilPodRunning` (60s, con su propio mensaje de error) llegó a dispararse antes de que vitest matara el test. Eso no calza con la flakiness de red ya documentada en ADR 0003 (propagación de CoreDNS, del orden de segundos, con reintentos que sí loguean) — apunta a algo más temprano, posiblemente el container `app` sin poder arrancar de verdad pese a que `pod.status.phase` reporte `Running` (Kubernetes no baja la fase a un pod que ya arrancó aunque el container entre en crash loop).

**Por qué se revirtió en vez de forzarlo:** sin acceso a un clúster K3s real para correr `kubectl describe pod`/`kubectl logs` durante el cuelgue, no hay forma de confirmar la causa exacta contra este entorno de desarrollo (Docker apagado, sin cluster local). Forzar un cambio de comportamiento de runtime sin poder verificarlo — en el repo de mayor superficie de seguridad del proyecto — no es un riesgo que valga la pena correr solo por cerrar el punto por completo.

**Confirmado, no solo sospechado:** el CI del PR corrió de nuevo tras sacar `readOnlyRootFilesystem` (mismo commit salvo esa línea) — el mismo test "camino feliz" que había colgado los 180s completos pasó en 16s. Es evidencia directa antes/después, no una atribución por descarte: la causa era efectivamente `readOnlyRootFilesystem`, no una flakiness genérica de CI.

**Propuesta para retomarlo:** (a) agregar logging intermedio al propio test (`waitUntilPodRunning` no loguea nada mientras espera, solo al fallar — un log cada 10-15s con la fase actual del pod haría el próximo intento diagnosticable desde el log de CI sin acceso al clúster); (b) o retomarlo con acceso real a un clúster (local con K3s/Docker levantado, o pidiéndole al owner correr el smoke test a mano) para inspeccionar el pod mientras cuelga.

**No bloquea nada:** el zip-slip (la parte crítica del punto 10) está cerrado sin depender de esto.

---

## Resumen para decidir

**Ronda 1** (puntos 1-9):

| # | Recomendación | Prioridad | Esfuerzo | Estado |
| --- | --- | --- | --- | --- |
| 1 | Readiness probe real | P0 | Bajo | ✅ PR #22 |
| 2 | Poda del historial de chat | P0 | Medio | ✅ PR #23 |
| 3 | `.env` en scripts standalone | P1 | Muy bajo | Pendiente (verificado sin cambios) |
| 4 | Colisión e2e / Redis local | P1 | Muy bajo | Pendiente (verificado sin cambios) |
| 5 | Heap de Node en tooling | P1 | Muy bajo | Pendiente (verificado sin cambios) |
| 6 | Playwright en CI de Jin_Web | P2 | Bajo | Pendiente |
| 7 | Reversión de acciones del agente | P2 | Alto (decisión) | Pendiente |
| 8 | Diagrama §2 desactualizado | P3 | Muy bajo | Pendiente |
| 9 | Iconos PWA reales | P3 | — (diseño) | Pendiente |

**Ronda 2** (puntos 10-27):

| # | Recomendación | Repo | Prioridad | Esfuerzo | Estado |
| --- | --- | --- | --- | --- | --- |
| 10 | Zip-slip en `files` + `readOnlyRootFilesystem` | Executor | P0 | Bajo | ✅ PR #6 |
| 11 | 3 notificaciones muertas esperando "Fase 2.4" | Core | P0 | Bajo | ✅ PR #24 |
| 12 | Lock del audit log no persiste a un restart | Core | P0 | Bajo | ✅ PR #24 |
| 13 | `audit_log` pierde actor/inputs externos al resolver | Core | P0 | Bajo | ✅ PR #24 |
| 14 | Contraseña de `jin login` en argv | CLI | P0 | Muy bajo | Handoff (Antigravity) |
| 15 | Redis y `memory.db` sin verificación de restore | Infra | P0 | Bajo | Handoff (Antigravity) |
| 16 | WS sin manejo de `disconnect` (UI colgada) | Web + CLI | P1 | Bajo | Pendiente (mitad Web es mía) |
| 17 | Mutaciones sin `catch` (aprobación fallida invisible) | Web | P1 | Bajo | Pendiente |
| 18 | `js-yaml` prod con advisory alto | Core | P1 | Muy bajo | Pendiente |
| 19 | `monaco-editor` → `dompurify` vulnerable | Web | P1 | Bajo | Pendiente |
| 20 | `test \|\| true` en CI; Web sin tests en CI | CLI + Web | P1 | Muy bajo | Handoff (mitad CLI) |
| 20.b | Tipos del CLI desincronizados: `jin tasks` no muestra actor/inputs externos | CLI | **P0** | Bajo (handoff) | Handoff (Antigravity) |
| 21 | Specs faltantes en piezas de seguridad | Core | P2 | Medio | Pendiente |
| 22 | Cero tests de cualquier tipo | Web | P2 | Medio | Pendiente |
| 23 | Cobertura mínima | CLI | P2 | Medio | Handoff (Antigravity) |
| 24 | Cablear probes al crear el Deployment | Infra | P3 | — (handoff) | Handoff (Fase 7.1) |
| 25 | `curl \| sh` sin checksum | Infra | P3 | Bajo | Handoff (Antigravity) |
| 26 | Init container sin `resources` | Executor | P3 | Muy bajo | ✅ PR #6 |
| 27 | Contrato OpenAPI de Executor desactualizado desde Fase 5.5 | Executor | P3 | Muy bajo | ✅ PR #6 |
| 28 | `readOnlyRootFilesystem` en preview-service — intentado, revertido por cuelgue en CI real | Executor | P1 | Medio (necesita clúster real) | Abierto |

**Mi recomendación de secuencia:**

0. **Handoff inmediato a Antigravity: 20.b.** Es el único punto donde hoy hay una brecha de información real en una pantalla de aprobación en producción-a-futuro, y el fix es mecánico (`pnpm generate:api` + mostrar dos campos). Va primero por ser barato y ajeno a mi área.
1. **Un PR de seguridad acotado: 10 + 14.** Son los dos explotables hoy con esfuerzo de fix bajo, y no dependen de ninguna decisión de política.
2. **Un PR de "el HITL avisa de verdad": 11 + 12 + 13.** Los tres son del mismo tejido (el sistema de aprobación/auditoría no le habla al humano cuando más importa) y comparten contexto, así que sale más barato juntos que separados. Es el bloque que más mejora la confiabilidad real del sistema.
3. **El PR de fricción de la Ronda 1: 3 + 4 + 5 + 20.** Media hora, cero riesgo, y el `|| true` de CI encaja acá porque es de la misma familia ("el tooling miente").
4. **15 antes de que 7.1 termine** — el backup no probado deja de ser teórico en cuanto haya datos reales.
5. **16 + 17** cuando se retome `Jin_Web`; son bugs de UX con consecuencia real en decisiones de aprobación.
6. **18 + 19** en el próximo ciclo de dependencias, juntos.
7. **21-23 (tests)** son deuda de fondo: valen más como política ("todo PR nuevo en estos repos trae su spec") que como un esfuerzo grande de recuperación de cobertura de una vez.
8. **24-26** con la Fase 7.1, que es cuando la infra se toca de verdad.
