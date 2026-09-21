# WEB_REDESIGN_V4_LIQUID_GLASS.md — Rediseño del dashboard de Jin

> **Qué es esto:** el prompt para rediseñar visualmente `Jin_Web` en [claude.ai/design](https://claude.ai/design).
> **Cómo se usa:** copia **todo lo que está debajo de la línea `═══`** y pégalo como primer mensaje.
> **Relación con `WEB_DESIGN_BRIEF.md` (v3, 2026-08-02):** aquel definió la *estructura* y sigue vigente en su lógica de producto. Este es un **rediseño visual (v4)** sobre una app ya construida y funcionando: cambia la piel y arregla lo que el uso real destapó, no reinventa el producto.
> **Fuente:** el código real de `Jin_Web` a 2026-09-21 (11 rutas + login, 4 componentes, 3 features). Cada "estado actual" de este documento está verificado en el código, no supuesto.
> **Estado:** escrito 2026-09-21, tras el primer uso real del dashboard contra la VM en producción.

═══════════════════════════════════════════════════════════

# Rediseña Jin — cabina de mando de un sistema de agentes de IA

## 0. Cómo quiero que trabajes

Esta app **ya existe y funciona**. No estás partiendo de cero: estás rediseñando la piel de un producto en uso. Eso cambia dos cosas:

1. **Respeta la estructura de información.** Las pantallas, los datos que muestran y los estados que manejan están decididos y probados. Cambia cómo se ven, no qué dicen.
2. **Hay invariantes de seguridad que el diseño NO puede romper.** Están en la sección 2. Léelas antes de tocar un color. Si una decisión estética choca con una de ellas, gana la invariante, y quiero que me lo digas explícitamente en vez de resolverlo en silencio.

Quiero **detalle ejecutable**: valores concretos (hex, px, ms, curvas), no adjetivos. Cuando digas "glass", quiero el `background`, el `backdrop-filter`, el `border`, la `box-shadow` y el `fallback`. Cuando digas "animación", quiero duración y curva.

---

## 1. Qué es el producto (contexto mínimo)

**Jin** es un sistema operativo personal que orquesta agentes de IA. Los agentes leen correo, gestionan calendario, siguen cursos universitarios, ejecutan código y levantan aplicaciones. **Ninguna acción con consecuencias reales ocurre sin que un humano la apruebe.**

- **Un solo usuario**, el dueño. No hay registro, roles, onboarding, equipos, facturación ni landing. Primera pantalla: login de una contraseña.
- **No es un SaaS.** No diseñes para convencer a nadie. Diseña para alguien que ya confía en la herramienta y la abre 20 veces al día, muchas desde el móvil.
- **Es una interfaz de confianza, no de productividad.** El trabajo lo hacen los agentes; el trabajo del humano es *decidir si dejarlos*. Todo gira alrededor de esa decisión.
- **PWA instalable**, tema oscuro único (sin modo claro, decisión cerrada).

---

## 2. Invariantes de seguridad — no negociables

Estas cinco reglas existen porque esta interfaz autoriza acciones irreversibles (enviar correos en nombre del dueño, borrar eventos futuros). Romperlas no es un problema estético: es un fallo de seguridad.

### 2.1. El ramp de riesgo es una escala ordinal validada
Hay 4 niveles de riesgo HITL, y su orden **se percibe** antes de leerse:

```
auto  <  notify  <  confirm  <  dual-confirm
```

Está implementado como un **ramp de un solo matiz** con luminosidad monótona (ΔL ≥ 0.06 entre pasos, spread de matiz ≤ 1°), validado contra el fondo. **Mantén esa propiedad**: si cambias los colores, el resultado debe seguir siendo un ramp ordinal del mismo matiz, no cuatro colores distintos. Un `dual-confirm` tiene que verse *más pesado* que un `confirm` aunque la pantalla esté en blanco y negro.

- `auto` **no lleva color**: es el escalón 0, deliberadamente casi invisible (solo texto atenuado). No le inventes un color "para que combine".
- **Nunca color solo.** La etiqueta de texto (`CONFIRM`, `DUAL-CONFIRM`) siempre acompaña al color. Daltonismo y modo alto contraste tienen que seguir funcionando.

### 2.2. El rojo saturado está reservado
Ver sección 3 — es el conflicto central de este rediseño.

### 2.3. La tarjeta `dual-confirm` es la más pesada de la interfaz
Es lo irreversible. Debe tener el borde más marcado, el mayor contraste y el botón de acción más prominente de toda la app. Nada decorativo puede competir con ella.

### 2.4. El botón de "Reanudar agentes" mantiene su fricción
Requiere **mantener pulsado 3 segundos**, con progreso visible. No lo conviertas en un click normal ni le quites el relleno de progreso. La fricción es el punto.

### 2.5. El contenido generado por IA va marcado
En la pantalla de Apps hay un aviso `⚠ GENERADO POR IA · AISLADO · NO CONFIABLE`. Ese aviso no se suaviza ni se esconde por estética.

---

## 3. El conflicto central: quiero un tema rojo, pero el rojo significa peligro

**Este es el problema de diseño más importante del encargo. Resuélvelo explícitamente.**

Hoy el rojo de marca (`#E2543F`) está **reservado** para dos cosas: el riesgo irreversible (`dual-confirm`) y el kill switch. Aparece poco, y por eso cuando aparece, se ve.

Quiero un tema **negro y rojo oscuro, tipo "liquid glass"**. Pero si toda la interfaz se vuelve roja, **el rojo deja de significar nada** y la señal de peligro se pierde en el ruido. Eso rompería la invariante 2.1 y 2.3.

**La solución que quiero que implementes: separa el rojo en dos roles distintos.**

| Rol | Dónde vive | Cómo se ve | Regla |
| --- | --- | --- | --- |
| **Rojo ambiente** (estructura) | fondos, tintes del cristal, bordes, sombras, degradados | **muy oscuro y desaturado** — casi negro con temperatura roja. Croma bajo (S ≤ 25%), luminosidad baja (L ≤ 12%) | Puede estar en todas partes. Nunca compite con el texto ni con una señal |
| **Rojo señal** (riesgo) | badge `dual-confirm`, borde de tarjeta dual, banner de kill switch, botón destructivo | **saturado y luminoso**, el actual `#E2543F` o más vivo | Aparece en ≤ 3 elementos por pantalla. Si aparece más, algo está mal |

La prueba que tiene que pasar tu diseño: **entrecierra los ojos en una captura de la bandeja de aprobaciones con un `dual-confirm` entre varios `confirm`. El dual tiene que saltar a la vista.** Si no salta, el rojo ambiente está demasiado vivo.

Dame en la entrega una **comparación lado a lado** de los dos rojos sobre el fondo real, con sus valores, para poder verificarlo.

---

## 4. El lenguaje visual: "liquid glass" sobre negro rojizo

### 4.1. La sensación que busco
Un panel de instrumentos oscuro, con capas de cristal translúcido que dejan intuir lo que hay detrás. Profundidad por **luz y desenfoque**, no por sombras duras. Bordes finos que capturan un reflejo. Calma: es una cabina, no una discoteca.

Referencias de sensación (no de marca): la translucidez de macOS/iOS moderno, el `frosted glass` de visionOS, paneles de instrumentos de coche de noche.

### 4.2. Reglas del cristal
Quiero que definas una **escala de elevación de 4 niveles**, y que cada componente diga en qué nivel vive:

- **Nivel 0 — Lienzo.** El fondo. Negro rojizo, opaco. Puede llevar un degradado radial muy sutil o un grano finísimo para que no se vea plano.
- **Nivel 1 — Cristal base.** Tarjetas normales. Translúcido, desenfoque medio, borde de 1px con luz arriba.
- **Nivel 2 — Cristal elevado.** Barras fijas (cabecera, navegación), elementos que flotan sobre contenido que hace scroll. Más desenfoque, más saturación del fondo.
- **Nivel 3 — Cristal crítico.** La tarjeta `dual-confirm` y los banners de emergencia. **Menos translúcido que el resto**: lo crítico se lee siempre, cueste lo que cueste.

Para cada nivel dame: `background` (con `color-mix` o `rgba`), `backdrop-filter` (blur + saturate), `border`, `box-shadow` (incluida la luz interior superior, el `inset` que da el efecto de borde de cristal) y el **fallback sin `backdrop-filter`**.

### 4.3. Los límites del cristal — importante
El cristal es bonito hasta que hace ilegible un dato. Reglas duras:

1. **Texto siempre sobre un fondo suficientemente opaco.** Ningún texto principal debe apoyarse solo en translucidez: si el fondo cambia, el contraste cae. Define una opacidad mínima de la capa detrás del texto y verifica contraste **AA (4.5:1)** para texto normal y **3:1** para texto grande y bordes de control.
2. **Nada de cristal sobre cristal sobre cristal.** Máximo dos capas translúcidas superpuestas. Más allá, el desenfoque se acumula y el rendimiento cae.
3. **Respeta `prefers-reduced-transparency`**: da una versión opaca completa, con los mismos colores sólidos.
4. **Respeta `prefers-reduced-motion`**: sin transiciones de entrada, sin parallax.
5. **`backdrop-filter` es caro.** No lo pongas en elementos que se repiten muchas veces (filas de tabla, ítems de lista largos). Reserva el desenfoque para superficies grandes y pocas: cabecera, navegación, tarjetas, modales.

### 4.4. Tipografía
Se mantienen las familias actuales (ya están cargadas): **IBM Plex Sans** para texto e **IBM Plex Mono** para datos.

La mono no es decorativa: marca **lo que es dato literal del sistema** (identificadores, hashes, nombres de herramienta, payloads, cifras). Esa distinción tiene que sobrevivir al rediseño.

Dame una escala tipográfica completa: tamaño, peso, interlineado y espaciado entre letras para: título de pantalla, título de tarjeta, cuerpo, cuerpo pequeño, etiqueta (las mayúsculas pequeñas tipo `ESPERANDO TU DECISIÓN`), dato mono y microcopia.

### 4.5. Movimiento
La regla actual, que quiero conservar: **el movimiento se reserva para lo que llega en vivo.** Una aprobación nueva que aparece puede animarse; un esqueleto de carga no debe brillar ni pulsar.

Define: duración y curva para entrada de elemento nuevo, cambio de estado de botón, transición entre pantallas (si la hay) y el relleno del botón de 3 segundos.

---

## 5. Layout y responsive

### 5.1. Puntos de ruptura
Hoy solo existe uno (720px) y se nota. Quiero tres:

| Rango | Nombre | Navegación | Contenido |
| --- | --- | --- | --- |
| < 720px | Móvil | Barra inferior fija | Una columna, ancho completo |
| 720–1100px | Tablet | Barra lateral estrecha, solo iconos, con tooltip | Una columna, ancho contenido |
| > 1100px | Escritorio | Barra lateral con iconos y texto | Una o dos columnas según pantalla |

### 5.2. La navegación móvil — el problema que más me molesta

**Estado actual, verificado en el código:** los ítems de la barra inferior son **enlaces de texto pelados**. Sin icono, sin fondo, sin píldora de activo. El único indicio de en qué pantalla estás es que el texto cambia de gris a blanco. El área táctil es de unos 20px de alto, muy por debajo del mínimo usable. Y **solo hay 5 de las 10 secciones**: Audit, Board, Editor, Apps y Memoria **no se pueden alcanzar desde el móvil**.

**Lo que quiero:**

1. **Iconos + texto corto** en cada ítem. Icono arriba, etiqueta debajo, ~10–11px.
2. **Estado activo inequívoco**: píldora de fondo con el cristal, icono relleno frente a contorno, color de texto pleno, y una marca superior (línea o punto). Que se vea de un vistazo y de reojo.
3. **Área táctil mínima de 44×44px real**, con la zona pulsable más grande que el dibujo.
4. **Las 10 secciones alcanzables.** Propón cómo: 5 fijas + un botón "Más" que abre una hoja inferior con el resto es mi apuesta, pero decide tú y justifícalo. Las fijas deberían ser las de uso diario: Overview, Aprobar, Chat, Gasto y Más.
5. **Indicador de pendientes**: si hay aprobaciones esperando, el ítem "Aprobar" lleva un contador. Es la razón principal por la que se abre la app.
6. **Respeta el área segura inferior** del iPhone (`env(safe-area-inset-bottom)`), ya contemplada pero conviene revisarla con la barra nueva.
7. **La barra es cristal nivel 2** y se mantiene fija sobre el contenido que hace scroll.

Dame el diseño de los tres estados de cada ítem: reposo, activo y pulsado.

### 5.3. Cabecera
Hoy: solo el texto `JIN` a la izquierda y el indicador de conexión a la derecha. Es poco. Quiero que siga siendo mínima, pero que el indicador de conexión tenga más presencia (es información de confianza: si no hay tiempo real, lo que ves puede estar desactualizado). En móvil debe poder colapsarse al hacer scroll para ganar altura.

### 5.4. Banners de emergencia
Hay dos que pueden aparecer sobre cualquier pantalla, apilados: **kill switch** (rojo señal, pleno) y **autonomía relajada** (aviso, no emergencia). Necesitan jerarquía distinta entre sí y no pueden empujar el contenido de forma que el usuario pierda el sitio. Define cómo conviven cuando se muestran los dos a la vez.

---

## 6. Primitivos

Diseña estos componentes con todos sus estados. Para cada uno: reposo, hover, foco (visible por teclado), activo/pulsado, deshabilitado y cargando.

1. **Tarjeta** — variantes: base, crítica (`dual-confirm`), y sutil (fondo hundido, se usa para el mensaje del usuario en el chat).
2. **Botón** — 4 variantes que ya existen: `default`, `primary`, `danger` (destructivo, contorno), `accent` (rojo señal, sólido). Altura mínima 44px en móvil. Incluye el botón especial de **pulsación sostenida de 3s** con su relleno de progreso.
3. **Badge de riesgo** — los 4 niveles del ramp.
4. **Campo de texto** — usado en login, chat y búsqueda de memoria. En móvil el tamaño de fuente debe ser ≥16px para que iOS no haga zoom al enfocar.
5. **Badge de conexión** — 4 estados: `EN VIVO`, `RECONECTANDO`, `SIN TIEMPO REAL`, `SIN CONEXIÓN`.
6. **Esqueleto de carga** — sin brillo animado (ver 4.5).
7. **Estado vacío** — título, detalle y acción opcional.
8. **Barra de progreso** — usada para el presupuesto del día.
9. **Ítem de navegación** — variantes de escritorio, tablet y móvil.
10. **Tabla / lista de datos** — ver pantalla de Audit.

---

## 7. Pantalla por pantalla

Para **cada una** quiero: composición en escritorio, composición en móvil, y los estados de carga, vacío y error. Si una pantalla necesita un tratamiento distinto en móvil (no solo más estrecho), dilo.

### 7.1. Login
**Qué hace:** una sola contraseña. Nada más.
**Ahora:** tarjeta centrada, 340px, con la etiqueta `SISTEMA PRIVADO`, el título `Jin`, un campo y un botón.
**Quiero:** que sea la primera impresión del lenguaje visual. Es la única pantalla donde puedes permitirte algo más atmosférico: un fondo con profundidad, el cristal bien visible. Debe transmitir "esto es privado y es serio", no "bienvenido a la app". Cuida el estado de error (`Contraseña incorrecta.`) y el de envío.

### 7.2. Overview — la pantalla de inicio
**Qué hace:** responde "¿necesitan algo de mí?" en menos de un segundo. Tres tarjetas: pendientes de decisión, presupuesto de hoy, actividad reciente.
**Ahora:** tres tarjetas iguales apiladas, todas con el mismo peso visual.
**Quiero:** jerarquía real. **"Esperando tu decisión" es la tarjeta principal** y debe dominar: si hay pendientes, es lo más llamativo de la pantalla, con el número grande y el botón de ir a la bandeja. Si no hay nada, se apaga y se vuelve discreta ("Nada esperando por ti"), cediendo protagonismo. Las otras dos son secundarias.
En escritorio ancho, considera dos columnas. En móvil, una sola, con la principal arriba.

### 7.3. Aprobaciones (`/hitl`) — **la pantalla más importante de la app**
**Qué hace:** aquí el dueño autoriza o rechaza acciones reales. Es donde el diseño importa de verdad.

Cada tarjeta muestra, en este orden: badge de riesgo + nombre de la herramienta + cuándo expira; el resumen del plan en lenguaje natural; quién lo pidió y qué contenido externo lo influyó; **el payload real** (los datos exactos de la acción, en mono, pares clave-valor); y los botones Rechazar / Aprobar.

**Detalles que no pueden perderse:**
- **"Influido por"** es información de seguridad: dice qué contenido externo (un correo, un anuncio) pudo haber inducido esta acción. Tiene que ser legible, no letra pequeña escondida.
- **El payload real** es la defensa contra que el resumen mienta. Debe ser cómodo de leer, incluso si es largo. Propón cómo tratarlo cuando tiene muchas claves o valores largos (¿plegable? ¿con scroll propio? decide y justifica).
- **Dual-confirm** tiene un flujo en dos pasos con una **espera forzada de 30 segundos** entre ambas aprobaciones. El botón cambia de texto: `Aprobar (1/2)` → `Espera 0 m` (con cuenta atrás que corre cada segundo) → `Confirmar (2/2)`. Diseña los tres estados y haz que la espera se *sienta* deliberada, no como un botón roto.
- **Mensaje de ejecución fallida**: cuando una aprobación se ejecutó y falló, la tarjeta lo dice y advierte que no se reintenta sola. Es un estado de alerta dentro de la tarjeta.
- **Estado "ejecutándose ahora"**: los botones se deshabilitan.
- **Proporción de los botones**: hoy Rechazar ocupa 1 parte y Aprobar 2. Mantén asimetría deliberada, pero cuida que Rechazar no quede tan pequeño que sea difícil de pulsar en móvil.

**En móvil:** es donde más se usa. Los dos botones deben ser cómodos con el pulgar y estar separados lo suficiente para no confundirlos. Considera si en móvil conviene que los botones queden fijos al fondo de la tarjeta visible.

### 7.4. Chat
**Qué hace:** se le pide un **objetivo** (no una tarea) y el agente arma un plan. La respuesta llega completa al final, **no hay escritura en vivo** (y eso se avisa).
**Ahora:** lista de turnos; el mensaje del usuario en una tarjeta hundida, la respuesta en una tarjeta normal, y entre ambas un componente de **plan** con pasos marcados `✓ … ○ ✕`.
**Quiero:**
- Distinguir claramente turno del usuario y respuesta del agente (hoy se distinguen poco).
- **El plan es el elemento distintivo de este producto**: muéstralo bien. Estados por paso: hecho, en curso, pendiente, fallido, con su nota opcional. Que se lea como un progreso real.
- Las **sugerencias** iniciales (dos botones con objetivos de ejemplo) merecen un tratamiento más atractivo que un botón gris.
- El texto de las respuestas puede venir con formato (negritas, listas, código). Define cómo se ve el contenido enriquecido dentro de la burbuja.
- **En móvil, el campo de entrada fijo abajo**, por encima de la barra de navegación, y que el teclado no lo tape. Ojo con `100dvh` y el teclado virtual de iOS.
- Estado "trabajando…" mientras el turno está abierto.

### 7.5. Gasto (`/budget`)
**Qué hace:** cuánto se ha gastado hoy, cuánto se lleva en la última hora comparado con lo normal, y el kill switch si está activo.
**Ahora:** tarjeta con el importe grande y barra de progreso; tarjeta de la última hora con el múltiplo (`2.4× lo normal`); y si el kill switch está activo, tarjeta crítica con el botón de 3 segundos.
**Quiero:** que el importe y el porcentaje sean lo primero. La barra de progreso debe cambiar de tratamiento al cruzar el 80% y el 100%. La tarjeta del kill switch activo es **cristal nivel 3** y debe verse como una emergencia. El botón de 3 segundos necesita un diseño de progreso claro y satisfactorio, y debe funcionar igual de bien con el dedo que con el ratón.

### 7.6. Autonomía
**Qué hace:** cambia cuánta libertad tiene el agente: `supervisado` (todo se aprueba), `semiautomático`, `automático`. **Bajar la protección exige doble aprobación**, y los modos relajados **caducan solos**.
**Ahora:** tarjeta de modo actual, dos tarjetas para activar cada modo relajado con un campo de horas, y una tarjeta con lo que sigue protegido.
**Quiero:**
- El modo actual debe verse como un **estado del sistema**, no como una opción más. Y debe ser evidente si está relajado, con **la cuenta atrás** de cuándo vuelve solo a seguro.
- Los tres modos merecen una representación visual de **escalón de riesgo creciente**, coherente con el ramp de la sección 2.1.
- Dejar clarísimo que pedir un modo relajado **no lo activa**: crea una aprobación que hay que confirmar dos veces. Hoy eso se dice en texto pequeño y se pierde.
- El botón "Volver al modo seguro ahora" es una salida de emergencia: siempre visible cuando aplica, fácil de encontrar.

### 7.7. Audit
**Qué hace:** la cadena inmutable de todo lo que pasó. Columnas: momento, nivel, herramienta, actor, estado, hash.
**Ahora:** una **tabla HTML con scroll horizontal**. En móvil es incómoda de verdad.
**Quiero:** en escritorio, tabla densa y legible (es para escanear muchas filas). **En móvil, no una tabla**: conviértela en lista de tarjetas compactas donde cada fila se lee sin scroll lateral. Define qué campos se ven en móvil y cuáles quedan en un detalle desplegable. El hash puede truncarse, pero debe poder copiarse. Mantén la paginación (`← Anterior` / `Siguiente →`).

### 7.8. Board de orquestación (`/orchestrator` y su detalle)
**Qué hace:** cuando el agente parte un objetivo en tickets, aquí se ve el tablero. Lista de ejecuciones, y al entrar, **columnas por estado**: `PENDIENTE`, `EN CURSO`, `HECHO`, `FALLIDO`, `BLOQUEADO`. Incluye detección de **conflictos abiertos** entre sub-agentes.
**Quiero:** en escritorio, columnas tipo kanban. **En móvil, las columnas no caben**: propón la alternativa (¿acordeón por estado? ¿pestañas? ¿scroll horizontal con indicadores?) y justifícala. Los **conflictos abiertos** son lo más importante de esta pantalla y deben destacar.

### 7.9. Apps corriendo (`/preview`)
**Qué hace:** lista de aplicaciones que los agentes levantaron, con su URL pública, cuánto les queda de vida y un botón de detener.
**Quiero:** el aviso `⚠ GENERADO POR IA · AISLADO · NO CONFIABLE` debe encabezar la lista y verse como una advertencia real (invariante 2.5). Cada entrada: URL, tiempo restante y botón destructivo. Trata visualmente el tiempo que se agota.

### 7.10. Memoria
**Qué hace:** búsqueda semántica de lo que el agente cree saber del dueño. Cada resultado tiene un tipo (`HECHO`, `PREFERENCIA`, `LECCIÓN`, `EPISODIO`), la cita textual, la fuente, la fecha y una distancia de similitud.
**Quiero:** que los cuatro tipos se distingan de un vistazo **sin usar el rojo señal**. La cita textual es el protagonista. La fuente y la fecha dan la trazabilidad ("de dónde lo sacó") y no deben desaparecer. Trata la búsqueda vacía como una invitación, no como un error.

### 7.11. Editor
**Qué hace:** un editor de código (Monaco) con comentarios de IA anclados a líneas concretas, cada uno con aceptar o descartar.
**Quiero:** un **tema de Monaco coherente** con el lenguaje visual (dame los colores de sintaxis sobre el fondo nuevo, cumpliendo contraste). El widget de comentario por línea debe integrarse con el editor sin taparlo. **En móvil, un editor de código es inusable**: propón qué mostrar (¿solo lectura? ¿un aviso de "mejor en escritorio"?) en vez de dejar que se rompa.

---

## 8. Accesibilidad — requisitos, no sugerencias

1. **Contraste AA** (4.5:1 texto normal, 3:1 texto grande y bordes de controles) sobre el fondo real, no sobre negro puro. Verifica especialmente el texto sobre cristal.
2. **Foco visible** en todo elemento interactivo, con un anillo que se vea sobre fondos translúcidos. No lo elimines por estética.
3. **Nunca solo color**: estados de riesgo, de conexión y de plan llevan texto o forma además del color.
4. **Objetivos táctiles ≥ 44×44px** en móvil.
5. **`prefers-reduced-motion`** y **`prefers-reduced-transparency`** respetados.
6. **Texto de campos ≥16px** en móvil (evita el zoom automático de iOS).
7. El contenido crítico (aprobaciones, kill switch) debe seguir siendo legible en **modo alto contraste** y con la interfaz en escala de grises.

---

## 9. Qué quiero recibir

1. **Sistema de tokens completo** en CSS: colores (con los dos rojos separados y justificados), superficies y niveles de cristal, espaciado, radios, tipografía, sombras, duraciones y curvas. Con el bloque de fallback para `prefers-reduced-transparency`.
2. **Los 10 primitivos** de la sección 6, con todos sus estados.
3. **Las 12 pantallas** de la sección 7, en escritorio y móvil, con estados de carga, vacío y error.
4. **La navegación móvil resuelta en detalle** (sección 5.2), incluido cómo se llega a las 10 secciones.
5. **La comparación de los dos rojos** sobre el fondo real (sección 3).
6. **Una nota de implementación** por pantalla: qué es puramente CSS y qué exige cambiar la estructura del componente. Me sirve para calcular el trabajo.

Si en algún punto crees que una petición mía choca con una invariante de la sección 2, **dímelo y propón la alternativa** en vez de elegir por mí.
