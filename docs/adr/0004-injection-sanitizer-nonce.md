# ADR 0004 — Sanitizador de injection: tag con nonce por sesión + escape HTML

## Contexto

`AGENTS.md` §5.1 exige que todo contenido de origen externo (correos, PDFs, páginas web, mensajes de Telegram entrantes, contenido de Canvas) que entra al contexto de un LLM se envuelva antes con una función dedicada. La versión original de esa sección ya especificaba el diseño final (tag `<untrusted_content_{sessionNonce}>` con nonce por sesión + escape HTML), pero citaba "ver ADR 0002" para su justificación — una referencia rota heredada del bundle de docs original: no existía ningún ADR todavía cuando se escribió esa cita, y cuando se creó ADR 0002 se usó ese número para un tema distinto (`request_id` de `audit_log`). Este ADR es el que esa cita debió apuntar desde el principio.

Construido durante Fase 3.1 (prerequisitos de Canvas LMS), antes de que existiera ningún consumidor real del módulo, siguiendo el mismo patrón de ADR 0002 y ADR 0003: documentar la decisión de diseño de seguridad antes/junto con el código, no después.

Un wrapper genérico sin nonce (`<untrusted_content source="...">...</untrusted_content>`) es vulnerable a un ataque simple: si el atacante controla parte del `content`, puede inyectar su propio `</untrusted_content>` dentro del payload para cerrar el tag antes de tiempo, seguido de texto arbitrario que el LLM interpretaría como fuera del bloque no confiable — efectivamente inyectando instrucciones que parecen venir del system prompt o de datos confiables.

## Decisión

1. **Cada sesión de agente genera un nonce de 16 caracteres hexadecimales** vía `generateSessionNonce()` (`Jin_Core/src/security/injection-sanitizer.ts`), usando `randomBytes(8).toString('hex')` — 8 bytes producen exactamente 16 caracteres hex; `randomBytes(16)` habría producido 32, incumpliendo la especificación de `AGENTS.md` §5.1.

2. **`wrapUntrustedContent(content, source, sessionNonce)` envuelve el contenido en `<untrusted_content_{sessionNonce} source="{source}">...</untrusted_content_{sessionNonce}>`.** El nonce forma parte del nombre del tag, no solo de un atributo — un atacante que inyecte `</untrusted_content_{sessionNonce}>` sin conocer el nonce exacto de la sesión activa no puede cerrar el tag real. La función:
   - Valida que `sessionNonce` matchee `/^[0-9a-f]{16}$/`; si no, lanza `InvalidSessionNonceError` (fail-safe: mejor fallar ruidosamente que envolver con un tag que el system prompt no reconocería como confiable).
   - Escapa `&`, `<`, `>` del contenido (en ese orden, para no escapar dos veces los `&` producidos por escapar `<`/`>`).
   - Escapa el mismo conjunto de caracteres más comillas dobles en el atributo `source`, para prevenir un escape del atributo HTML además del escape del tag.

3. **`sanitizeForIndexing(content)` aplica el mismo escapado sin el tag**, para el segundo uso que exige `AGENTS.md` §5.1 punto 2 (sanitizar antes de indexar en pgvector o en la memoria sqlite-vec). No tiene consumidor todavía — `src/memory/**` no existe (llega en una fase posterior) — se deja lista para cuando exista, en vez de omitirla, porque es un requisito explícito del documento y no una anticipación especulativa.

## Consecuencias

- El escape HTML por sí solo ya neutraliza el ataque de cierre-de-tag descrito arriba (el `<` del cierre fabricado queda literal, no se interpreta como inicio de tag). El nonce agrega una segunda capa independiente: aunque el escape fallara por un bug, un atacante sin el nonce de la sesión activa no puede fabricar un cierre válido.
- `sanitizeForIndexing` queda sin consumidor real hasta que se construya `src/memory/**` — deuda técnica explícita y deliberada, documentada aquí para que no se pierda, no un olvido.
- El contrato de "qué va en `source`" (nombre de la integración: `"canvas"`, `"gmail"`, `"telegram"`, etc.) no está formalizado como un enum — queda como string libre por ahora; si el número de integraciones crece lo suficiente para que valga la pena, se puede tipar más estrictamente en un PR futuro sin romper el contrato externo de la función.

## Alternativas consideradas

- **Tag genérico sin nonce (`<untrusted_content source="...">`):** es exactamente el diseño vulnerable descrito en el Contexto — rechazado.
- **Nonce en un atributo (`<untrusted_content nonce="{sessionNonce}" source="...">`) en vez de en el nombre del tag:** un atacante todavía podría fabricar `</untrusted_content>` sin necesitar adivinar el nonce, porque el nombre del tag de cierre no depende de él — el escape HTML seguiría neutralizándolo, pero se perdería la segunda capa de defensa independiente que motiva tener nonce. Rechazada.
- **Nonce sin escape HTML:** un atacante que lograra filtrar o adivinar el nonce de la sesión (ej. por un log mal configurado) podría fabricar un cierre válido; el escape HTML es necesario incluso con nonce, no es redundante. Rechazada como diseño único — se implementan ambas capas.
- **Validar el nonce con una librería de esquemas (Zod) en vez de una regex simple:** para un valor tan simple (string de longitud y alfabeto fijos) una regex es suficiente y no amerita la dependencia adicional en este módulo específico; el resto de la config del repo sí usa Zod donde la forma es más compleja (`AGENTS.md` §8.4).
