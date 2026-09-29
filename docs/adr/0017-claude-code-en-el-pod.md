# ADR 0017 — Claude Code dentro del pod, con la suscripción del owner

## Contexto

El owner (2026-09-29) quiere correr **Claude Code dentro de los pods de terminal**, usando **su suscripción** (Pro/Max) y **no una API de pago**. Hasta ahora eso no se podía por dos razones, y las dos son a propósito (ADR 0016):

- el pod **no tiene internet** (solo el proxy de npm del clúster);
- el pod **no tiene secretos**: no recibe variables de entorno ni tokens, así que un `postinstall` malicioso no tiene nada que robar.

Claude Code necesita alcanzar los servidores de Anthropic y una credencial. Abrir eso **cambia el modelo de amenaza**; este ADR dice cómo, y qué riesgo queda.

## Cómo se autentica una suscripción en un servidor

`claude setup-token` (se corre **una vez en el Mac del owner**; aprueba en el navegador) imprime un token OAuth de un año atado a la suscripción. En el pod se usa como `CLAUDE_CODE_OAUTH_TOKEN`. El uso cuenta contra los límites de la suscripción, como en el Mac.

## Decisión

```
pod (claude, HTTPS_PROXY) ──CONNECT api.anthropic.com:443──▶ claude-egress (ns claude-egress) ──TCP──▶ api.anthropic.com
   └─ solo con jin.io/claude=enabled                          └─ allowlist de dominios, sin descifrar
```

1. **Opt-in por sesión y con aprobación.** El interruptor "Habilitar Claude Code" al abrir el pod manda `claudeCode: true`; la aprobación es `confirm` fijo (ningún modo de autonomía la relaja, el modelo no puede pedirla) y su resumen dice sin ambigüedad: *el pod podrá salir a los servidores de Anthropic y guardar tu token; un código malicioso dentro del pod podría leerlo*. Sin la bandera el pod es exactamente el de antes.
2. **Un proxy propio que solo deja llegar a Anthropic** (`Jin_Infra/k8s/base/claude-egress`): un túnel HTTP `CONNECT` de ~150 líneas, sin dependencias, que **no descifra el tráfico** (no ve el token ni la conversación). Solo `host:443` de una allowlist (`api.anthropic.com`, `claude.ai`, `console.anthropic.com`; se ajusta con la primera prueba real); IP literales, otros puertos y todo lo que no sea `CONNECT` se rechazan. Resuelve el DNS él mismo y rechaza cualquier dominio que resuelva a una dirección privada, loopback, link-local (incluido `169.254.169.254`) o CGNAT, y conecta a la IP ya verificada (sin ventana de DNS rebinding). Topes de túneles y de vida. Solo registra host, bytes y duración.
3. **Red en capas.** `NetworkPolicy` `default-deny` en `claude-egress`; su ingress acepta solo pods `jin.io/type=terminal` **y** `jin.io/claude=enabled`; su egress es DNS y `443` sin ninguna dirección interna. Y la política **por pod** que crea el Executor suma una única regla hacia el proxy, solo con la bandera.
4. **El token no viaja con el pod y nunca pasa por la terminal.** El owner lo pega en la app ("Conectar Claude Code") y se guarda en el disco del proyecto (`.home/.claude-token`) por el explorador de archivos, cuyo audit registra la **ruta** y nunca el contenido. La terminal interactiva lo exporta como `CLAUDE_CODE_OAUTH_TOKEN` al abrirse. No es una línea tecleada, así que no queda en el audit; y como defensa en profundidad, la vista previa de cada línea que sí se audita **redacta** `sk-ant-…` y las asignaciones de `CLAUDE_CODE_OAUTH_TOKEN` / `ANTHROPIC_API_KEY` / `ANTHROPIC_AUTH_TOKEN` **antes** de cortar a 120 caracteres.
5. **Claude Code sin ruido:** sin telemetría ni autoupdate (`CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC`, `DISABLE_AUTOUPDATER`); instalado con `npm i -g` en el disco del proyecto (`npm_config_prefix`), vía el proxy de npm existente; `HOME` en el disco, para que su configuración persista. `HTTPS_PROXY` con `NO_PROXY` para el proxy de npm y el clúster.

## Riesgo residual (aceptado por el owner, 2026-09-29)

- **El token queda dentro del pod.** Cualquier código que corra ahí (un `postinstall` de un paquete de npm, o el propio proyecto) puede leerlo y usarlo. Con una clave de API en un proxy que la inyecta el pod no la vería nunca; con la suscripción, sí. Si un token se filtra, alguien puede usar la suscripción del owner (con sus límites) hasta que se revoque.
- **Mitigaciones:** opt-in por sesión con aprobación explícita; el token dura un año y se revoca desde la cuenta de Claude ("Desconectar" en la app borra el archivo del pod pero **no lo revoca**); la salida del pod solo llega a Anthropic (un token robado no se puede sacar a otro destino por este canal; el canal de exfiltración de bajo ancho de banda del proxy de npm que ya aceptaba el ADR 0016 sigue existiendo); el proxy no puede leer el tráfico.
- **Términos de uso.** Anthropic limita el uso de tokens de suscripción a sus propias herramientas; acá corre Claude Code de Anthropic, sin modificar, hablando directo con sus servidores. Lo que **no** se hace es autenticar herramientas de terceros con el token.

## Alternativas descartadas

- **Clave de API de pago.** El owner no quiere pagar por API.
- **Un proxy que inyecte el token** (el pod nunca lo vería). No hay garantía de que Claude Code acepte un token de suscripción "de relleno" ni de que reenviarlo con un proxy propio respete los términos de Anthropic. Podría reconsiderarse si Anthropic documenta un modo de gateway para suscripciones.
- **Abrir la salida por CIDR de Anthropic.** Las IPs cambian y una `NetworkPolicy` no filtra por dominio (ADR 0003): por eso un proxy con allowlist de dominios.
- **Iniciar sesión con `/login` dentro del pod.** Necesita completar un flujo de navegador desde un servidor sin navegador y tiene problemas conocidos en entornos remotos; `setup-token` es el camino pensado para eso.

## Consecuencias

- **Jin_Infra:** namespace `claude-egress` (proxy + `NetworkPolicy`), sin Secrets (no hay credenciales en el clúster).
- **Jin_Executor:** `claudeCode` en abrir/reanudar → label, variables de entorno, regla de egress; `.home`/`.npm-global` fuera de "Traer archivos"; la terminal exporta el token si existe el archivo.
- **Jin_Core:** `claudeCode` en la aprobación (resumen sin ambigüedad) y redacción de secretos en el audit.
- **Jin_iOS:** interruptor "Habilitar Claude Code", hoja "Conectar Claude Code" (token en un `SecureField`, guardar/desconectar, instalar), insignia en la terminal.
- **No verificado hasta desplegar y probar con el token real:** que la allowlist de dominios alcance para que Claude Code arranque y responda (`api.anthropic.com`, ¿`claude.ai`?, ¿otros?), y que `HTTPS_PROXY` funcione con `CLAUDE_CODE_OAUTH_TOKEN`.

## Fuera de esta tanda

- Que Jin (el chat) use Claude Code del pod.
- Rotar el token automáticamente, o guardarlo cifrado (hoy vive en el disco del proyecto).
- Ver o cerrar sesiones de Claude Code desde la app.
