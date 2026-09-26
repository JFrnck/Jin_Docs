# ADR 0013 — Cliente iOS nativo (`Jin_iOS`): Bearer, Socket.IO propio y cero dependencias

## Contexto

El owner usaba el dashboard web como PWA en su iPhone 16 Pro Max y tuvo problemas reales:

- el nav de abajo no quedaba fijo;
- el chat perdía los mensajes al cambiar de pestaña;
- Jin no tenía contexto de lo dicho antes;
- no se veía qué modelo respondía;
- "Jin no respondía", sin ningún progreso visible.

Se escribió `docs/IOS_DESIGN_BRIEF.md`. El owner lo llevó a Claude Design y el resultado quedó en `Jin/design_app_ios/` (sistema "Nocturne").

Aprobado por el owner el 2026-09-25, con tres decisiones suyas:

1. La tarjeta de aprobación es **lista compacta → detalle a pantalla completa**. La espera del dual-confirm es un número grande con candado.
2. **App primero, servidor después.** Lo que el diseño pide y el servidor no entrega se construye igual, marcado "PRÓXIMAMENTE".
3. **Inter + JetBrains Mono empaquetadas, íconos SF Symbols.**

## Decisiones

### Repo propio, lo lidera Claude Code

`Jin_iOS` va aparte, igual que Jin_Web y Jin_CLI: un consumidor más del contrato de Jin_Core.

Adentro tiene:

- un proyecto Xcode (carpetas sincronizadas, así que sumar archivos no toca el `.pbxproj`);
- un paquete local `JinKit` con dos productos:
  - `JinKit`: lógica sin UI, testeable;
  - `JinUI`: sistema de diseño y pantallas.

### Autenticación: el mismo Bearer que la CLI, cero cambios de servidor

`POST /api/auth/login` ya devuelve `accessToken` en el body. El token:

- se guarda en el Keychain (`WhenUnlockedThisDeviceOnly`);
- viaja como `Authorization: Bearer` por HTTP;
- viaja como `auth.token` en cada CONNECT de Socket.IO, una forma que `extractWsToken` ya aceptaba.

No hay cookies, así que no hay CORS ni CSRF que resolver en un cliente nativo.

El `exp` del JWT se lee solo para mostrar el vencimiento ("sesión vence el 2 oct"), nunca para decidir. Quien decide es el servidor:

- Un 401 en cualquier llamada limpia el token y vuelve al login.
- Si el 401 llegó durante una acción, el login dice explícitamente qué **NO** se ejecutó.

### Socket.IO propio sobre `URLSessionWebSocketTask`

No se agregan dependencias de terceros (AGENTS.md §1.1). Un paquete de terceros habría sido la única dependencia de la app entera, y además en el camino por donde llegan las aprobaciones.

El cliente implementa lo mínimo que usa Jin_Core:

- **Engine.IO v4**: open, ping del servidor → pong y mensaje.
- **Socket.IO v5**: CONNECT con `auth`, EVENT, DISCONNECT y CONNECT_ERROR.
- Los namespaces `/` y `/chat` multiplexados en un solo WebSocket.
- Reconexión con backoff de 1 a 8 s, igual que la web.
- Un watchdog: si no llega ping en `pingInterval + pingTimeout`, la conexión se da por muerta.

El parser es de funciones puras y tiene tests.

**Verificado en vivo contra Jin_Core local** (socket.io 4.8.3, 2026-09-26), usando las fuentes de `JinKit` tal cual:

- se une a `/` y `/chat`;
- sigue en vivo después de más de un ciclo de ping/pong;
- un turno de `chat:message` termina en `chat:error` cuando los dos modelos fallan;
- el servidor expulsa un token inválido de ambos namespaces;
- el login devuelve `wrongPassword` con los intentos restantes y el 401 se mapea a `unauthorized`.

### Las reglas de seguridad se aplican en los stores, no solo en la UI

- **Sin conexión viva no se decide nada.** Aprobar, rechazar, reanudar el kill switch y cambiar la autonomía quedan deshabilitados. Nada queda en cola para "cuando vuelva la red": una decisión diferida es una decisión tomada sin ver el estado actual.
- **Nada se decide deslizando.** Solo con botones explícitos.
- **La espera del dual-confirm sale de `availableAt` del servidor**, nunca de un reloj local. Después de aprobar se vuelve a pedir la lista, no se infiere el estado.
- **El 2/2 pide Face ID o el código del dispositivo** (`.deviceOwnerAuthentication`). Si la biometría no está disponible, falla cerrado. Viene activado y se puede apagar en Ajustes.
- **Reanudar el kill switch es mantener pulsado 3 s.**
- **Ninguna notificación ni widget tiene botón de decisión.** Todavía no existen; la regla queda escrita para cuando existan.
- **El payload de una aprobación se muestra en el orden de claves del servidor.** `JSONValue` tiene un parser propio porque `JSONSerialization` desordena las claves. Lo que el owner lee tiene que ser exactamente lo que se va a ejecutar.

### Chat: el contexto viaja con cada turno

Las conversaciones se guardan en el dispositivo, en un JSON con `.completeFileProtection`, en un store que vive en la raíz de la app. Así, cambiar de pestaña no pierde nada.

Cada turno manda `history`:

- el texto del usuario;
- el `finalResponse`;
- y, si el servidor la devolvió, la `compactedHistory`, que **reemplaza** al historial local en vez de sumarse.

Con esto se arregla el "Jin no tiene contexto".

Con Jin_Core #50/#51 desplegados, la app muestra en vivo el plan, las tools y el texto vía `chat:progress`, más el modelo que respondió. Sin ellos degrada a solo la respuesta final, sin romperse.

### Lo que falta del lado del servidor se ve, pero marcado

Se construye con su diseño y la etiqueta "PRÓXIMAMENTE":

- recordatorios de Canvas/Gmail;
- tokens por turno;
- gasto por hora y por modelo;
- contador del freno de autonomía;
- estado de la sesión de Claude Code;
- estado de la cadena y resultado de la ejecución en Audit;
- notificaciones push.

Esos endpoints van en una tanda aparte en Jin_Core, cada uno con su PR.

## Consecuencias

- **Un tercer cliente del contrato.** Un cambio que rompa `contracts/openapi.json` ahora también rompe la app. Los modelos Codable de `JinKit` se escribieron a mano contra el contrato y hay tests de decodificación con fixtures. Por ahora no hay generador de tipos para Swift.
- **El cliente Socket.IO es código nuestro que mantener.** Si Jin_Core sube de versión mayor de socket.io, hay que repetir la prueba en vivo antes de desplegar.
- **Sin firma ni distribución todavía.** El owner instala desde Xcode con su Apple ID. TestFlight y APNs requieren la cuenta de desarrollador de pago y quedan para otro ADR.
- **Fuera de este ADR:** la extensión de widgets y Live Activity / Dynamic Island, que necesita un target nuevo y un App Group, y mandar `history` también desde Jin_Web, que tiene la misma causa del "no tiene contexto".
