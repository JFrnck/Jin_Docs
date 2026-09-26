# ADR 0014 — Notificaciones push (APNs), construidas y apagadas hasta tener la cuenta Apple Developer

## Contexto

El diseño de la app iOS (`design_app_ios/Jin Sistema.dc.html` §9 y §10) pide:

- avisos de aprobaciones, sin botones de decidir;
- responder a Claude Code desde la notificación;
- resumen matutino;
- Live Activities que se actualicen con la app cerrada.

Todo eso necesita APNs, y APNs necesita la cuenta Apple Developer de pago. Con el Personal Team gratuito, Xcode rechaza el permiso: *"Personal development teams do not support the Push Notifications capability"*.

Pedido del owner (2026-09-26): **dejarlo listo para usar apenas se pague**. Plan aprobado el mismo día.

## Decisiones

### Construido entero y apagado por configuración, no a medias

- **Servidor:** `APNS_KEY_ID`, `APNS_TEAM_ID` y `APNS_PRIVATE_KEY` son opcionales, con la regla "las tres o ninguna" (mismo patrón que el puente, ADR 0012). Sin ellas, `PushService` loguea "Push apagado" y los eventos se ignoran. Así Jin nunca queda en CrashLoop por una función que todavía no se puede usar.
- **App:** build setting `JIN_PUSH` (`NO` por defecto) que elige los entitlements (`Jin.entitlements` o `Jin-Push.entitlements`, con `aps-environment`) y el flag `JinPushEnabled`. El código Swift se compila siempre, así que no se pudre esperando.
- Activarlo es el runbook `docs/runbooks/apns-activation.md`: clave en Infisical, reiniciar jin-core, `JIN_PUSH = YES` y reinstalar.

### Cliente APNs propio, sin dependencias

`node:http2` (una sesión por entorno, reutilizada) y un JWT ES256 firmado con `node:crypto`, con `dsaEncoding: 'ieee-p1363'` porque JOSE exige `r||s` y Node firma en DER por defecto. El JWT se cachea 50 min: Apple rechaza tokens de más de 1 h y castiga renovarlos muy seguido. Un 410 o `BadDeviceToken` borra el token.

Con AGENTS.md §1.1 en mente, una librería de terceros para ~150 líneas bien testeadas no se justificaba.

### Push solo avisa

- Las aprobaciones llegan con la categoría `JIN_APPROVAL`, **sin acciones**: tocar abre la tarjeta. Lo fijan tests en ambos lados. Incluso una acción de respuesta inyectada sobre un aviso de aprobación se ignora.
- `PushModule` no importa `HitlModule` ni `AuditModule`, y no escribe en `audit_log`.
- La única acción es "Responder" (y un botón por opción) en los avisos de Claude Code: es mensajería del puente (ADR 0012), no HITL. Exige desbloquear el iPhone (`.authenticationRequired`).
- El texto del agente (`planSummary`) se trunca, como en Telegram.

### Qué se avisa

Hay ocho tipos, uno por toggle de Ajustes de la app, con los mismos ids en los dos lados:

- aprobación nueva y por vencer;
- acción notify ejecutada (apagada por defecto);
- presupuesto 80 % y 100 %;
- kill switch;
- mensaje de Claude Code;
- resumen matutino;
- turno de chat terminado.

Este último solo se avisa si la app lo pidió (`notifyWhenDone`) y el socket ya no está.

### Live Activities por push

- **Orquestación:** evento nuevo `orchestration.run.changed`, emitido en los puntos de mutación de `OrchestratorService`.
- **Kill switch:** push-to-start, iOS 17.2+.
- **`content-state`:** se arma en TypeScript igual que en Swift. Las fechas van en segundos desde 2001, porque ActivityKit usa el `JSONDecoder` por defecto.
- Para que las dos implementaciones no se desincronicen, la fixture `Jin_Core/src/push/__fixtures__/orchestration-content-state.json` se valida en Jest y se decodifica en un test de Swift.

### Un solo sondeo del presupuesto

`BudgetAlertMonitor` (`src/budget/`) reemplaza el sondeo que tenía `RealtimeGateway` y emite eventos que consumen el gateway y push. Con push eran tres consumidores, el umbral de AGENTS.md §1.1. `TelegramBotService` conserva el suyo: su deduplicación es distinta y quedó fuera de este cambio.

## Consecuencias

- **Migración 0014:** tablas `push_devices` y `push_activity_tokens`. Existen aunque push esté apagado; con push apagado quedan vacías.
- **Contrato:** se agregan `/api/push/*` y `notifyWhenDone` (opcional) en el chat. Solo agrega.
- **No verificado:** el envío real a Apple, hasta tener la clave. Se probó el cliente contra un servidor HTTP/2 local, y la app en el simulador: permiso, token y registro contra un Jin_Core local, y un aviso de aprobación que abre esa aprobación.
- **No accionables en el simulador:** el botón "Enviar" de la respuesta y la extensión de notificaciones; están cubiertos por tests.
- **Bugs encontrados antes de llegar al iPhone:**
  - crash al tocar un aviso con el delegate `async`;
  - respuesta desde el aviso perdida, porque la app se lanza sin interfaz;
  - tokens de 80 y 128 bytes, más largos que el tope inicial.
- **Pendiente para cuando haya cuenta:** TestFlight en lugar de reinstalar cada 7 días. El mismo pago lo habilita, con otro runbook.
