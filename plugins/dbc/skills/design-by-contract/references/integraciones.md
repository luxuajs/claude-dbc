# Integraciones: correo, pagos, archivos y hosting

> **Precios verificados el 2026-10-01.** Las capas gratuitas cambian a menudo (en 2025–2026
> cambiaron AWS, Polar y SES). Si han pasado más de 3 meses, **verifica con WebSearch antes
> de recomendar** y actualiza esta tabla.

## Índice
1. Reglas comunes · 2. Correo transaccional · 3. Pagos · 4. Almacenamiento de archivos ·
5. Hosting · 6. Cómo registrar cada decisión

---

## 1. Reglas comunes a toda integración

1. **El contrato habla nuestro idioma, no el del proveedor.** Las entidades son nuestras
   (`Pago`, `Suscripcion`, `Archivo`) con nuestros estados. El único sitio donde aparece el
   proveedor es la ruta de su webhook (`/v1/webhooks/pagos/stripe`), porque ese cuerpo lo
   define él. Así cambiar de proveedor es escribir otro adaptador, no reescribir el contrato.
2. **Un adaptador por proveedor** detrás de una función nuestra, elegido por variable de
   entorno (`CORREO_PROVEEDOR=resend`). Vive en `backend/src/integraciones/<tipo>/` (pequeño)
   o `backend/packages/integraciones/` (mediano+). No crees la interfaz "por si acaso" si
   solo hay un proveedor: basta con una función `enviarCorreo()`; el segundo proveedor
   llega cuando haga falta.
3. **Webhooks entrantes:** antes de escribirlos en el contrato, mira en la documentación del
   proveedor qué manda de verdad: cabecera de firma (`Stripe-Signature`,
   `X-Twilio-Signature`…), content-type (Twilio manda formularios y espera TwiML) y qué
   respuesta espera. La firma se verifica sobre el cuerpo **en crudo**, antes de parsear. La
   idempotencia sale de una clave **natural** (el id del evento), declarada con
   `x-clave-idempotencia`; nunca de una `Idempotency-Key` que el proveedor no manda. Si el
   contrato usa un dialecto neutral, `x-adaptador` dice qué adaptador traduce antes de
   validar. Respuesta 2xx rápida y el trabajo pesado después.
4. **Secretos solo en variables de entorno**, con su nombre en `.env.example` sin valor.
5. **Modo prueba en desarrollo** (claves *test*/*sandbox*), nunca claves reales en local.
6. **Topes de gasto** configurados en todo proveedor de pago por uso, y un runbook que diga
   dónde están.

---

## 2. Correo transaccional

| | Resend | AWS SES |
|---|---|---|
| Gratis | 3.000/mes, **máx. 100/día**, 3 dominios | Cuentas nuevas (desde 15-jul-2025): **sin capa gratis propia**; se paga con los créditos de AWS ($100–200, 6 meses) |
| Después | Pro $20/mes | ~$0,10 por 1.000 correos |
| A favor | Mejor experiencia de desarrollo, React Email, panel claro | El más barato a volumen; natural si ya estás en AWS |
| En contra | El tope de 100/día se alcanza antes que el mensual | Empieza en *sandbox*: hay que pedir acceso a producción; consola más compleja |

**Recomendación por defecto:** Resend para validar el MVP. Pasar a SES cuando el volumen
haga que los $20/mes de Resend pesen (decenas de miles de correos al mes) o cuando el
proyecto ya viva en AWS. Si el MVP envía muchos correos en un mismo día (lanzamientos,
invitaciones masivas), el tope de 100/día de Resend es el primer problema.

**Implementación:**
- Una función `enviarCorreo({ para, plantilla, datos, claveIdempotencia })`. Plantillas en
  código (React Email sirve para ambos proveedores).
- Dominio propio verificado con **SPF, DKIM y DMARC**, idealmente en un subdominio
  (`mail.midominio.com`) para no arriesgar la reputación del dominio principal.
- Enviar **después** de confirmar la transacción en base de datos, con reintento. Si un
  fallo del proveedor no debe perder correos, tabla *outbox* + reintento periódico (añádela
  cuando ese riesgo sea real, no antes).
- Rebotes y quejas: webhook del proveedor → marcar la dirección como no válida. Obligatorio
  antes de enviar a volumen (SES suspende cuentas con muchos rebotes).
- En el contrato el correo no aparece como tal: aparecen las operaciones que lo disparan
  (p. ej. `POST /v1/auth/recuperar` → 202).

Variables: `CORREO_PROVEEDOR`, `CORREO_REMITENTE`, `RESEND_API_KEY` | `AWS_REGION` + credenciales.

---

## 3. Pagos

### Elegir

| Situación | Elige | Por qué |
|---|---|---|
| SaaS o producto digital vendido a todo el mundo, sin empresa en un país de Stripe, o sin ganas de declarar IVA/sales tax en cada país | **Polar.sh** | *Merchant of Record*: Polar es el vendedor legal y se encarga de los impuestos. Starter gratis con 5% + 50¢ por transacción (las organizaciones creadas antes del 27-may-2026 mantienen 4% + 40¢) |
| Tienes empresa en un país soportado por Stripe; e-commerce físico; o quieres control total | **Stripe** | El estándar: Checkout, Billing (suscripciones), Customer Portal. Los impuestos son responsabilidad tuya (Stripe Tax aparte). Disponibilidad por país: stripe.com/global |
| Clientes en Latinoamérica que pagan en moneda local (PSE, OXXO, Pix, efectivo, cuotas) | **Mercado Pago** | Disponible en AR, BR, CL, CO, MX, PE, UY. Comisión aprox. 3,49%–4,99% + IVA según país. Checkout Pro (redirige) para empezar; Suscripciones para cobros recurrentes |
| Clientes que quieren pagar con cripto | **Binance Pay** | **Solo como complemento**, nunca como único medio. Exige alta de comercio (KYB). Pagos entre billeteras Binance sin coste; *payouts* 0,8% con tope de 5 USD |

Combinaciones habituales: Polar o Stripe para tarjetas internacionales + Mercado Pago para
LatAm. Polar no sirve para bienes físicos: si es e-commerce de productos físicos, Stripe y/o
Mercado Pago.

### Implementación (igual para todos)

- **Nunca toques datos de tarjeta.** Usa el checkout alojado del proveedor (redirección o
  componente embebido). Así el alcance PCI es el mínimo.
- **La verdad llega por webhook, no por la redirección.** La página de "gracias" no confirma
  nada: el pago se da por hecho cuando llega el webhook firmado.
- Guarda el **id del evento** del proveedor y no proceses dos veces el mismo (los proveedores
  reintentan).
- **Dinero como texto decimal + moneda** (`{ "importe": "19.90", "moneda": "USD" }`), nunca
  como número de coma flotante.
- Contrato neutral, por ejemplo:
  - `POST /v1/pagos/checkout` → `{ url }` (el backend crea la sesión con el proveedor elegido)
  - `GET /v1/suscripciones/{id}` → estado nuestro: `pendiente | activa | vencida | cancelada`
  - `POST /v1/webhooks/pagos/{proveedor}` — uno por proveedor, firmado e idempotente
- El acceso a funciones de pago se decide con **nuestro** estado de suscripción, no
  consultando al proveedor en cada petición.
- Más adelante (no en el MVP): conciliación periódica proveedor ↔ base de datos.

Variables: `PAGOS_PROVEEDOR`, y por proveedor su clave secreta + secreto del webhook.

---

## 4. Almacenamiento de archivos

| | Cloudflare R2 | DigitalOcean Spaces | AWS S3 |
|---|---|---|---|
| Gratis | **10 GB/mes y salida de datos siempre gratis** | No | Solo con los créditos de cuenta nueva ($100–200, 6 meses) |
| Después | Por GB y por operación | **$5/mes fijo**: 250 GiB + 1 TiB de salida + CDN | Por GB + **salida de datos** (lo que más sorprende en la factura) |
| A favor | $0 real para un MVP | Precio plano y predecible | El ecosistema más completo |

Los tres hablan la **API de S3**: un solo adaptador con `@aws-sdk/client-s3` y la variable
`ARCHIVOS_ENDPOINT`. Cambiar de proveedor = cambiar variables de entorno.

**Recomendación por defecto:** R2 para validar ($0), Spaces si prefieres precio fijo y CDN
incluido, S3 si el proyecto ya está en AWS.

**Implementación:**
- **URLs prefirmadas.** El cliente pide permiso (`POST /v1/archivos/enlace-subida` → `{ url,
  clave }`), sube **directo al bucket** y confirma (`POST /v1/archivos/{clave}/confirmar`).
  Los archivos grandes nunca pasan por el backend (los planes gratis y serverless tienen
  límites de tamaño y tiempo).
- Al prefirmar, limita **tipo y tamaño** (política POST con `content-length-range`).
- **Buckets privados.** Descargas con URL prefirmada de vida corta. Público solo lo que de
  verdad lo es (avatares, imágenes de catálogo), en un bucket aparte.
- Claves con estructura: `<tenant o usuario>/<tipo>/<uuid>.<ext>`. Los metadatos (dueño,
  nombre original, tamaño) van en la base de datos; nunca se lista el bucket para buscar.
- CORS del bucket configurado para el dominio del frontend.

Variables: `ARCHIVOS_ENDPOINT`, `ARCHIVOS_REGION`, `ARCHIVOS_BUCKET`, `ARCHIVOS_ACCESS_KEY`, `ARCHIVOS_SECRET_KEY`.

---

## 5. Hosting

Se decide **por fase**. Pregunta en qué fase está el proyecto.

| | Vercel | Render | Railway |
|---|---|---|---|
| Gratis | Hobby: **solo uso personal, no comercial**. Cobrar, captar clientes o anunciar un servicio cuenta como comercial | Servicios web gratis para siempre, **se duermen a los 15 min** sin tráfico (~1 min en despertar), 750 h/mes. **Postgres gratis caduca a los 30 días** (+14 de gracia, luego se borra) | Prueba de 30 días con $5 de crédito (pide tarjeta); después $1/mes de crédito: no da para algo encendido 24/7 |
| Pago | Pro $20/mes por asiento + consumo | Desde ~$7/mes por servicio | Hobby $5/mes con $5 de consumo incluido |
| Mejor para | Frontend Next.js | Backend y sitios estáticos | Backend siempre encendido y barato; varios servicios |

**Base de datos:** no uses el Postgres gratis de Render salvo para una demo de días. Para
validar, usa un Postgres gratis que no caduque (Neon o Supabase, plan gratis) y
`pg_dump` diario, porque los planes gratis no tienen recuperación a un punto en el tiempo.

### Por fase

| Fase | Frontend | Backend | Base de datos | Coste aprox. |
|---|---|---|---|---|
| **Validación** (demo, sin cobrar) | Vercel Hobby o sitio estático en Render | Render gratis (acepta que se duerme) | Neon o Supabase gratis | $0 |
| **Primer cliente que paga** | Vercel Pro, o estático en Render (gratis) | Railway Hobby o Render de pago (siempre encendido) | Neon o Supabase gratis o su primer plan de pago | ~$5–30/mes |
| **Crecimiento** | Vercel Pro | Railway Pro / Render / contenedores propios | Plan de pago con copias de seguridad y recuperación a un punto en el tiempo | según uso |

Señales para salir de la fase de validación: el primer cliente que paga (Vercel Hobby deja
de valer), webhooks de pagos o de terceros (un backend dormido tarda ~1 min en responder y
el proveedor puede dar el webhook por fallido), o la base de datos se acerca a su límite.

**Reglas para poder mudarse sin dolor:**
- Un `Dockerfile` por servicio desplegable, aunque la plataforma no lo exija.
- Toda la configuración por variables de entorno; nada atado a una plataforma en el código.
- `/salud` en cada servicio para las sondas de la plataforma.
- Topes de gasto activos (gestión de gasto en Vercel, presupuestos en AWS, límites en Railway).
- Backend y base de datos en la misma región.

---

## 6. Cómo registrar cada decisión

Por cada integración elegida:
1. **ADR** `docs/decisions/NNNN-<tipo>-<proveedor>.md`: qué se eligió, qué alternativas se
   descartaron y **cuándo se revisará** (p. ej. "al pasar de 2.000 correos/mes").
2. Fila en la tabla de piezas de `docs/architecture.md` (pieza, proveedor, plan, coste).
3. Variables en `.env.example`.
4. Operaciones en el contrato si las hay (checkout, enlace de subida, webhooks) — §D.
5. Tareas en la cola: adaptador, webhook, verificación del dominio o bucket, runbook de topes
   de gasto. Cada una con "Leer" y "Hecho cuando".

Por cada integración **descartada**, una línea en `docs/architecture.md`, en "No usa", para
que ninguna sesión futura la añada por iniciativa propia.
