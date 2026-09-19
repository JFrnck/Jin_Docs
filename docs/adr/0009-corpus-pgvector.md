# ADR 0009 — Corpus propio en pgvector

## Contexto

`BLUEPRINT.md` §3.3.1 traza una frontera explícita entre dos almacenes de vectores que nunca debían mezclarse: "pgvector = corpus (correos indexados, PDFs de Canvas, notas), grande, con JOINs relacionales" vs. "sqlite-vec = memoria del agente (hechos, preferencias, episodios), pequeña, curada". El segundo ya estaba construido (Fase 4.3, `src/memory/`); el primero no existía — la imagen de Postgres ya trae pgvector y la extensión ya se crea en el primer arranque del clúster (`Jin_Infra/k8s/base/postgres/init-configmap.yaml`), pero ningún módulo la usaba. Fase 9.3.

## Decisiones

### 1. Fuente indexada: correos (Gmail), no PDFs de Canvas ni notas

BLUEPRINT declara tres fuentes; la tarea pide implementar de verdad la de mayor valor y documentar el resto como extensión, sin stubs vacíos. Gmail ya tiene toda la plomería real (`GoogleGmailClientService`, tool `readEmails`, OAuth funcionando desde Fase 4.2) — cero trabajo previo pendiente. PDFs de Canvas necesitarían una librería de extracción de texto nueva (ninguna en el repo). Notas ni existen como fuente todavía (dependen de Notion, Fase 9.1, sin código). Extensión futura: agregar una fuente nueva es, en este diseño, solo un nuevo caller de `CorpusService.indexEmail`-equivalente con un `source` distinto — la tabla `corpus_entries` ya es agnóstica de fuente (`source`/`sourceId`/`metadata` libres).

### 2. Dos tablas (`corpus_entries` + `corpus_embeddings`), nunca una sola denormalizada

Es el argumento entero de §3.3 para elegir pgvector sobre Qdrant/Supabase: "un JOIN entre tasks y task_embeddings es SQL nativo, no una sincronización entre dos servicios". Colapsar todo en una sola tabla con una columna `vector` habría sido más simple de escribir, pero no habría demostrado ni ejercitado esa ventaja real — el JOIN en `CorpusService.search()` es exactamente ese caso de uso, no un adorno.

### 3. `vector()` nativo de drizzle-orm, sin paquete `pgvector` externo

`drizzle-orm@0.45.2` ya expone `vector(name, { dimensions })` en `drizzle-orm/pg-core` (mapea `number[]` ↔ el formato de texto `[v1,v2,...]` que pgvector acepta). El paquete npm `pgvector` (con su propio helper de Drizzle) quedó descartado — sería una dependencia nueva para algo que el ORM ya cubre.

### 4. Dedup real a nivel de base de datos, no solo en el código

`corpus_entries` tiene un índice UNIQUE en `(source, source_id)`; `corpus_embeddings` tiene un índice UNIQUE en `entry_id` (1 embedding por entrada, nunca 1:muchos). `CorpusService.indexEmail()` hace `ON CONFLICT ... DO UPDATE` contra ambos — reindexar el mismo correo actualiza en el lugar, nunca acumula filas huérfanas que ensuciarían la búsqueda con resultados duplicados del mismo contenido.

### 5. `rag_hit_ratio`: hit = ≥1 resultado, no un umbral de distancia

BLUEPRINT §10.1 lista la métrica sin definir "hit" con precisión. Se descartó un umbral de distancia coseno (ej. "hit si distancia < 0.3") por ser un número arbitrario sin evidencia real que lo respalde todavía — la definición elegida ("la búsqueda encontró algo vs. el corpus no tenía nada relevante que devolver") es la lectura más directa y verificable de "ratio de aciertos de la búsqueda", y no bloquea agregar un umbral de relevancia más adelante si la operación real lo pide.

### 6. Hallazgo real corregido de paso: imagen de Postgres de los tests de integración

`test/support/postgres-testcontainer.ts` (compartido por TODOS los integration tests del repo) pinneaba `postgres:16.14-alpine` — sin el binario de la extensión `vector` — con el comentario explícito "No usa pgvector aquí". Cualquier test de esta fase habría fallado al crear la extensión. Se actualizó al mismo `pgvector/pgvector:0.8.5-pg16` que ya usa producción y `docker-compose.dev.yaml` — superset estricto de la imagen oficial de Postgres 16, sin impacto esperado en los tests de integración existentes (verificado corriendo la suite completa, no solo el archivo nuevo).

## Consecuencias

- Agregar una segunda fuente (PDFs de Canvas, notas) requiere una librería de extracción de contenido nueva por fuente, pero ningún cambio de esquema — `corpus_entries`/`corpus_embeddings` ya son agnósticas.
- HNSW (pedido explícito de §3.3) sobre `corpus_embeddings.embedding`. IVFFlat queda documentado como fallback si la RAM del clúster aprieta, sin implementarlo — mismo criterio que ya aplicó el owner a PgBouncer/Tempo (diferido hasta que exista una señal real, no antes).
- **Nota de numeración de migraciones:** esta fase (PR de Jin_Core) y Fase 9.5 (feature flags, PR #32) reservaron ambas una migración nueva en paralelo, en ramas distintas, contra la misma cadena de snapshots de drizzle-kit ya rota desde la 0003 (deuda documentada, ver STATUS.md Fase 9.5). Fase 9.5 usó `0008`; esta fase usó `0009` a propósito, asumiendo que 9.5 se mergea primero. Si el orden de merge se invierte, quien mergee la que quede segunda va a necesitar renumerarla — señalado explícito para que no se pierda.

## Alternativas consideradas

- **Una sola tabla denormalizada (metadata + vector juntos):** rechazada, ver punto 2 — no demuestra el argumento real de §3.3.
- **Paquete `pgvector` npm para el tipo de columna:** rechazado, ver punto 3 — dependencia nueva innecesaria.
- **Indexar automáticamente cada `readEmails`, sin una tool separada:** se consideró, pero mezclar un side-effect de escritura dentro de una tool `auto` de solo lectura ya establecida (`readEmails`) habría sido una sorpresa de comportamiento no declarada explícitamente al LLM ni al owner. Una tool separada (`indexEmailToCorpus`) deja la decisión de qué vale la pena indexar explícita en el tool-calling, auditable como cualquier otra acción del agente.
