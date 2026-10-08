# ¿El contrato está listo para implementar?

Un contrato puede pasar el lint, el bundle y el humo y aun así no poderse implementar: la
forma es válida pero el contenido obliga a inventar. Esta lista se recorre **contra los
fragmentos** (`contracts/<servicio>/`, nunca el bundle) antes de repartir trabajo a backend y
frontend. Cada "no" es una tarea de contrato.

## 1. Webhooks y proveedores (el fallo más caro)

- [ ] Cada webhook describe lo que **de verdad** manda el proveedor, comprobado en su
      documentación: cabecera de firma (`Stripe-Signature`, `X-Twilio-Signature`…),
      content-type (Twilio manda formularios y espera TwiML) y respuesta que espera.
- [ ] Ningún webhook exige `Idempotency-Key`: los proveedores no la mandan. La clave es
      **natural** y se declara: `x-clave-idempotencia: 'cuerpo:/id'` o `'cabecera:<nombre>'`.
- [ ] Si el contrato habla un dialecto neutral, cada webhook declara su `x-adaptador` y
      `info.description` explica que el cargador verifica la firma sobre el cuerpo **en crudo**
      y traduce **antes** de validar. Si no, el cargador rechaza el payload real con 422.
- [ ] Un esquema de seguridad por proveedor, con su cabecera real.
- [ ] Si el proveedor resuelve algo en una sola petición (p. ej. el webhook de entrada de un
      motor de voz que acepta o rechaza), el contrato no lo parte en dos operaciones.

## 2. Cada flujo descrito tiene su operación

- [ ] Cada valor de un `enum` de estado tiene una operación (o un proceso declarado) que
      lleva a él. Un `cancelada` sin "cancelar" es un estado inalcanzable.
- [ ] Cada "se avisa", "se envía", "vence", "se borra a los N días" tiene **quién lo
      dispara**: operación de usuario, interna, o programada (cola con retardo / cron) con
      su esquema de seguridad.
- [ ] Cada cobro tiene cómo pagar (enlace de checkout) y cómo gestionarlo (portal).
- [ ] Cada dato que una descripción da por existente tiene de dónde salir (p. ej. "franjas
      libres" → ¿de qué agenda?).
- [ ] Lo que un servicio necesita de otro está como ruta interna (`/v1/internal/*`), no
      como lectura de su base de datos.

## 3. Sesión, roles y rutas públicas

- [ ] Se sabe quién emite la sesión y qué claims trae (`sub`, `rol`, `tenant_id`…).
- [ ] Las reglas de acceso están escritas una vez en `info.description`: recurso de otro
      tenant → 404; rol insuficiente → 403 (`x-roles`); listados del operador con su
      parámetro de tenant, ignorado para el resto.
- [ ] Las rutas públicas (`security: []`) declaran su protección: captcha (Turnstile) y
      límite por IP (`x-limite-peticiones` → 429).

## 4. Esquemas sin ambigüedad

- [ ] Los cuerpos de `PUT`/`POST` no exigen campos que genera el servidor (`id`, `version`,
      fechas): esquema de entrada propio.
- [ ] `type: [X, 'null']` con `enum` incluye `null` en el `enum` (si no, nunca valida).
- [ ] Nada de `additionalProperties: true` en datos que alguien tiene que interpretar:
      argumentos por tipo con `oneOf` + discriminante obligatorio.
- [ ] Ninguna descripción contradice al esquema ("si está vacío…" con `minItems: 1`).
- [ ] Transiciones de estado escritas (cuáles valen y qué responde lo demás: 409).
- [ ] Cupos y límites de negocio con su respuesta (p. ej. 409 al sexto "fundador").
- [ ] Rangos raros representables (horario que cruza la medianoche).

## 5. Respuestas y errores

- [ ] `401` en toda operación con credencial, `403` en las del panel, `500` en todas, `429`
      donde haya límite. Lo que el servidor puede responder y el contrato no declara saldrá
      como violación en `prism proxy`.
- [ ] Catálogo de `type` de error en `common/errores.yml` (incluidos los que ya use el código).

## 6. Cierre

- [ ] `grep -rn "x-pendiente" contracts/ --include=*.yml` vacío.
- [ ] Lint **sin avisos**, `pnpm typecheck`, `pnpm humo`.
- [ ] El mock responde a todas las operaciones: un script que recorre las operaciones del
      bundle y llama a cada una contra `pnpm mock` (cuenta como respuesta cualquier código
      que no sea "ruta/método no encontrado" ni 5xx).
- [ ] Los tipos viven en `contracts/` y el frontend no depende de ningún paquete de `backend/`.
- [ ] Versión subida (mayor si hubo cambios incompatibles), entrada de cierre en
      `CAMBIOS.md` y tag `contrato-v<versión>`.

Con todo marcado, la tarea siguiente es **derivar las tareas** de backend y frontend del
contrato cerrado y comprobar, con un script, que cada `operationId` queda en exactamente una
tarea de backend y en una pantalla o en la lista "sin pantalla" con su motivo.
