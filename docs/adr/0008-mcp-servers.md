# ADR 0008 — MCP servers para documentación externa

## Contexto

BLUEPRINT §6.4: "documentación técnica externa (React, Tailwind, Next.js, etc.): MCP servers oficiales (Context7, etc.). No mantenemos pipeline propio de scraping." Fase 7.3, verificado en la auditoría de seguridad (`docs/security-audit-fase7.md`) que no existía código MCP en ningún repo.

El resultado de una tool MCP es contenido de una fuente externa como cualquier otra (AGENTS.md 5.1) — la pregunta de diseño real no es "cómo conectar el SDK", sino cómo evitar que esta integración se convierta en una superficie de ataque nueva.

## Decisiones

### 1. Una sola tool estática y reviewed en el registry, nunca las tools que el servidor liste dinámicamente

La alternativa obvia — descubrir las tools que expone cada servidor MCP vía `listTools()` y registrarlas dinámicamente como tools reales del agente — se descartó explícitamente: eso es **capability injection**. `AGENTS.md` 5.4 exige que el `hitlLevel` de cada tool sea estático y revisado por PR; si el *conjunto de tools disponibles* también viniera de un servidor remoto, un MCP server comprometido (o uno legítimo que agregue una tool nueva sin que nadie lo note) podría exponer una operación mutante bajo un nombre inocente y el agente la llamaría con `hitlLevel: 'auto'` sin que ningún humano la haya visto nunca.

En vez de eso: **una sola tool** (`queryExternalDocs`, `Jin_Core/src/tools/registry.ts`), estática, con `hitlLevel: 'auto'` fijado por PR. Su implementación (`src/mcp/mcp-client.service.ts`) conoce exactamente 2 operaciones concretas del servidor Context7 (`resolve-library-id` + `get-library-docs`, documentadas públicamente) y **nunca** proxea un nombre de tool elegido por el LLM hacia el servidor remoto. `listTools()` sí se llama, pero solo para **verificar** que esas 2 tools existan antes de usarlas — nunca para decidir qué se puede invocar.

### 2. El wrapping de contenido externo se hereda gratis del pipeline existente, sin tocar `agent.service.ts`

`stringifyToolResult` + `wrapUntrustedContent` (`AgentService.handleRealToolCall`) ya envuelven el resultado de **toda** tool ejecutada, sin importar su origen. Como el executor de `queryExternalDocs` (`src/mcp/mcp.module.ts`) devuelve un string plano, ese mismo código — sin ninguna modificación — ya lo trata como contenido externo no confiable. No hacía falta abrir `agent.service.ts` para esto: es exactamente el resultado que se buscaba, logrado reusando el mecanismo ya auditado en la Fase 8.2 (golden set) en vez de inventar un segundo punto de wrapping.

### 3. Conexión perezosa, nunca al arranque de la app

`config/mcp-servers.yaml` se carga al boot (mismo patrón que `models.yaml`), pero la conexión real al servidor MCP (`Client.connect()`) es perezosa — recién en el primer uso de `queryExternalDocs`, con la conexión cacheada y descartada del cache si falla (para que una caída transitoria no "envenene" el proceso completo). Un servidor de documentación caído es, por diseño, un problema de baja severidad (`hitlLevel: 'auto'`, sin efecto en ninguna otra funcionalidad) — no debe poder tumbar el arranque de `jin-core` como sí lo haría, correctamente, una falla de Postgres o de Infisical.

### 4. Parsing best-effort del formato de Context7, documentado como tal

`resolve-library-id` devuelve texto libre, no JSON estructurado; extraer el ID de librería requiere un regex contra el formato público documentado por Context7 (`- Context7-compatible library ID: /org/project`). **No verificado contra el servidor real en este entorno** (sin acceso de red al implementarlo) — si el formato cambia, `McpUnexpectedResponseError`/`McpUnsupportedServerError` lo hacen explícito (AGENTS.md 1.4: fail rápido y ruidoso, nunca un resultado vacío silencioso). Queda como recomendación probarlo manualmente contra el servidor real antes de considerar esta integración 100% validada en producción.

## Consecuencias

- Agregar un segundo servidor MCP (u otra librería con otro formato de resolución) requiere código nuevo en `McpClientService` — no es un mecanismo genérico "cualquier servidor MCP funciona automáticamente". Deliberado: la alternativa genérica es exactamente el riesgo de capability injection del punto 1.
- `@modelcontextprotocol/sdk` es el único cliente MCP en el repo — sin pipeline propio de scraping, tal como pide BLUEPRINT §6.4.
- Tests con el SDK mockeado (`mcp-client.service.spec.ts`), mismo criterio que cualquier SDK externo (AGENTS.md 6.3) — ninguno habla con Context7 real.

## Alternativas consideradas

- **Registro dinámico de tools MCP descubiertas:** rechazado, ver punto 1 (capability injection).
- **Proxy genérico `callMcpTool(server, toolName, args)` expuesto al LLM:** mismo problema que la alternativa anterior con una capa menos de indirección — el LLM elegiría `toolName` libremente. Rechazado.
- **Wrapping manual dentro de `McpClientService`** (en vez de heredarlo del pipeline de `agent.service.ts`): habría duplicado la lógica de nonce/escapado fuera de su único lugar de verdad (`injection-sanitizer.ts`) sin necesidad. Rechazado.
