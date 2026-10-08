---
name: design-by-contract
description: >-
  Monta y opera proyectos design by contract: un openapi.yml es la fuente de verdad y backend, frontend y CLI se construyen contra él, en un monorepo (contracts/, backend/, frontend/, database/, docs/) que escala de pequeño a grande. Incluye flujo bidireccional (si el código no puede cumplir el contrato, se cambia primero el contrato), puertas de CI, ADR, cola de tareas en PROGRESO.md, planes por tarea para que un modelo planifique y otro implemente con pocos tokens, comprobación de que el contrato está completo antes de implementar, y decisiones de correo transaccional (Resend, SES), pagos (Polar, Stripe, Mercado Pago, Binance Pay), archivos (S3, Spaces, R2) y hosting barato para MVP (Vercel, Render, Railway). Úsala al iniciar un proyecto, adoptar la metodología en código existente, cambiar el contrato, auditar un repo o un contrato, planificar o implementar una tarea, añadir pagos/correo/archivos/CLI, elegir hosting, o cuando digan "sigue con la siguiente tarea", "qué falta", "retoma donde quedamos", "planifica", "implementa el plan", aunque no mencionen OpenAPI.
---

# Design by contract

El contrato (`contracts/openapi.yml`) **es** el sistema. El backend carga sus rutas y su
validación desde él, el frontend usa tipos generados de él, y nada se programa si el contrato
no lo declara. Un contrato que miente es peor que no tener contrato, y una sesión que relee
todo el repo es tiempo y dinero tirados.

## Primero: ¿en qué modo estoy?

Lo que se hace en cada sesión tiene su propio comando, más barato que esta skill:

| Comando | Para qué |
|---|---|
| `/dbc:plan <ID> [<ID>…]` | Escribir planes (`plans/<ID>.md`) que otro modelo implementa (fija Opus) |
| `/dbc:implement <ID>` | Implementar un plan leyendo solo el plan (fija Sonnet) |
| `/dbc:next [carril]` | Avanzar la siguiente tarea de la cola cuando no tiene plan |

Esta skill queda para lo demás. Palabras clave al inicio de los argumentos: `contrato
<cambio>` → §D · `auditoría` o `completitud` → §E · `integraciones` → §I.

Sin palabra clave, decide mirando el repo (`ls`; ¿existe `PROGRESO.md`?, ¿y `contracts/`?):

| Situación | Modo |
|---|---|
| Carpeta vacía o idea nueva | **A · Proyecto desde cero** |
| Hay código pero no `contracts/` o no `PROGRESO.md` raíz | **B · Adoptar en código existente** |
| Existe `PROGRESO.md` y el usuario quiere avanzar | Indícale `/dbc:next` (o `/dbc:implement <ID>` si hay plan) y aplícalo |
| Añadir o cambiar un endpoint, campo o error | **D** |
| Pregunta si su estructura o su contrato están bien | **E** |
| Añadir correo, pagos, archivos, hosting o CLI | **I** |

Si sigue sin estar claro, pregunta una sola cosa: "¿empezamos de cero, adaptamos lo que hay,
o seguimos con la cola?".

## Principios que atraviesan todos los modos

1. **El contrato va primero.** El código nunca va por delante. Si algo no es viable, **se
   detiene el código**, se cambia el contrato (con su porqué) y después se sigue.
2. **Contrato completo antes de repartir trabajo.** Las tareas de backend y frontend se
   derivan del contrato **cerrado**, no de uno a medias: así cada lado avanza por separado sin
   inventar (`references/completitud.md` dice cuándo está cerrado).
3. **El contrato se hace cumplir, no se promete.** El backend no arranca si sus manejadores y
   el contrato no coinciden; el frontend no compila si los tipos cambian; la CI no deja hacer
   merge si el contrato cambió sin regenerar o sin explicarse.
4. **Lo generado no se edita ni se lee** (`contracts/dist/`, `contracts/generated/`). Cuesta
   decenas de miles de tokens y no aporta nada que no esté en los fragmentos.
5. **Una tarea por sesión.** La cola (`PROGRESO.md`) dice qué va; el plan (`plans/<ID>.md`),
   si existe, dice cómo. Cada tarea dice qué leer y cuándo está hecha.
6. **Empieza pequeño.** Crea solo el mínimo del tamaño del proyecto; lo demás entra como tarea
   cuando hace falta. Una plantilla vacía "por si acaso" es ruido que cada sesión lee.
7. **Nunca sobrescribas un archivo sin leerlo.** En proyectos existentes, fusiona.

Stack por defecto (cámbialo si el usuario o el repo usan otro): TypeScript, Node ≥ 22, pnpm
workspaces, Fastify (o Express), Next.js como **cliente** del contrato (sin Server Actions ni
rutas API haciendo de backend), Prisma, PostgreSQL. Herramientas: `@redocly/cli`,
`openapi-typescript`, `openapi-fetch`, `@stoplight/prism-cli`, `oasdiff`. Prioriza siempre
el menor coste de infraestructura: suelen ser MVP.

## Referencias

Léelas solo cuando el modo lo pida y **solo la sección que toca**: localízala con
`grep -n "^## " references/<archivo>` (en `plantillas.md`, `grep -n "^## [0-9]"`, porque sus
plantillas también tienen títulos) y lee ese tramo, no el archivo entero.

- `estructura.md` — árbol estándar, tamaños, qué documento responde qué.
- `estilo.md` — SOLID, DRY, KISS, patrones, comentarios y OWASP Top 10:2025 en este stack.
- `completitud.md` — ¿el contrato está listo para implementar? Lista de comprobación.
- `flujo-y-puertas.md` — flujo bidireccional, las 7 puertas, CI, cambios incompatibles.
- `plantillas.md` — CLAUDE.md, cola, PROGRESO de carpeta/servicio, ADR, CAMBIOS.md,
  settings.json, scripts, openapi.yml raíz, errores, runbook, PR, **plan de tarea (§17)**.
- `backend-contrato.md` — el backend carga rutas del contrato y no arranca si hay deriva.
- `integraciones.md` — correo, pagos, archivos, hosting y webhooks: qué elegir y cómo.
- `cli.md` — CLI como consumidor del contrato.

---

## §A · Proyecto desde cero

1. **Entiende la idea** en 3–5 preguntas: qué hace, quién lo usa, qué dominios tiene,
   cuántos servicios hacen falta *hoy* (casi siempre uno), qué terceros llaman por webhook.
   Decide el **tamaño** con `estructura.md` § Tamaños; ante la duda, pequeño. Después
   **pregunta por las integraciones** (§I paso 1); no supongas ninguna.
2. **Crea el esqueleto mínimo** (`estructura.md` + `plantillas.md`): `CLAUDE.md`,
   `README.md`, `PROGRESO.md`, `.claude/settings.json`, `package.json`,
   `pnpm-workspace.yaml`, `.gitignore`, `docs/architecture.md` (con su **Stack**),
   `docs/decisions/0001-design-by-contract.md`, `contracts/` (openapi.yml, common/,
   redocly.yaml, CAMBIOS.md, PROGRESO.md, package.json con los tipos) y los `PROGRESO.md` de
   `backend/`, `frontend/`, `database/`. Sin `runbooks/`, `.github/`, `plans/` ni `tools/`
   vacíos: entran cuando hacen falta.
3. **Escribe el contrato v0.1.0 con el usuario**, dominio por dominio, con las reglas de
   `plantillas.md` §13 explicadas en `info.description`: RFC 9457 en `common/`; `operationId`
   en toda operación; el primer `tag` nombra el servicio dueño; `Idempotency-Key` en las
   escrituras del cliente que cuesten dinero; **webhooks con la forma real del proveedor**
   (cabecera de firma, content-type, respuesta esperada; compruébalo en su documentación) y
   clave de idempotencia **natural** (`x-clave-idempotencia`), nunca una cabecera que el
   proveedor no manda; dinero como texto decimal + moneda; 3.1 sin `nullable`; identidad y
   tenant desde la sesión, con los roles declarados (`x-roles`).
4. **Enchufa el contrato**: `pnpm contrato` en verde; backend con el cargador de
   `backend-contrato.md` y un manejador por `operationId` que devuelve 501; humo que demuestra
   el freno de deriva; `pnpm mock` para el frontend.
5. **Llena la cola con el tronco de contrato**, no con el reparto: tareas para completar el
   contrato hasta que pase `completitud.md`, una de cierre (versión + tag) y una final
   **"derivar las tareas de backend y frontend del contrato cerrado"**, que es la que crea los
   dos carriles (backend por grupos pequeños de `operationId`, frontend por pantalla). Cada
   tarea con "Leer" y "Hecho cuando".
6. Diagramas si hacen falta: C4 con `c4-arquitectura`, procesos con `bpmn-negocio`.
7. Termina diciendo cuál es la primera tarea de la cola.

## §B · Adoptar en código existente

El objetivo es llegar a "el contrato manda" **sin reescribir** el proyecto.

1. **Inventario barato**: árbol (sin `node_modules`), `package.json`, framework, dónde se
   declaran las rutas (`grep -rn "app\.\(get\|post\|put\|patch\|delete\)\|router\." --include=*.ts`),
   OpenAPI existente (`find . -name "*openapi*" -o -name "*swagger*"`), base de datos y ORM,
   integraciones presentes (`grep -rlE "resend|@aws-sdk|stripe|polar|mercadopago|binance" --include=*.ts --include=package.json .`).
   No leas archivos enteros. Las integraciones que existan se respetan; las que falten, §I.
2. **Contrato de línea base**: si hay OpenAPI, muévelo a `contracts/` y pártelo por dominio;
   si no, **genéralo de lo que el código hace hoy** (la única vez que el contrato sale del
   código). Versión `0.x` y en `CAMBIOS.md`: "línea base extraída del código el <fecha>; desde
   aquí el contrato manda". Lo dudoso, con `x-pendiente: <duda>`, nunca inventado.
3. **Reorganiza solo lo necesario**, con `git mv`. Si toca más de un par de carpetas,
   propónlo antes. No cambies el framework.
4. **Enchufa en modo aviso**: valida contra el contrato y registra la deriva sin bloquear.
   Cada discrepancia es una tarea ("ajustar contrato" o "ajustar código", decidido con el
   usuario). Con cero discrepancias, modo bloqueo.
5. **Crea o fusiona** `CLAUDE.md`, la cola, los `PROGRESO.md` de carpeta,
   `.claude/settings.json` y `docs/architecture.md` (Stack sacado del código real).
6. **ADR 0001** con la adopción e informe corto: qué hay, qué cambió, qué falta (la cola).

## §C · Sesión de trabajo → `/dbc:next`

Los pasos viven en `${CLAUDE_PLUGIN_ROOT}/skills/next/SKILL.md`. Si la tarea tiene plan, en
`${CLAUDE_PLUGIN_ROOT}/skills/implement/SKILL.md`. Si te invocaron así, lee ese archivo y aplícalo.

## §D · Cambio de contrato

1. Edita el fragmento (`contracts/<servicio>/` o `contracts/common/`). Nunca `dist/` ni tipos.
2. `pnpm contrato` hasta que el lint quede **sin avisos**.
3. ¿Incompatible? (quitar endpoint o campo, hacer obligatorio un opcional, estrechar un
   `enum`, cambiar tipo o significado). Si sí: decídelo con el usuario y escribe un ADR; con
   consumidores externos, `/v2` en vez de romper `/v1`.
4. Entrada en `contracts/CAMBIOS.md`: fecha, versión, qué cambió y **por qué la
   implementación no podía cumplir lo anterior**. Sube `info.version` (SemVer). Durante el
   tronco de contrato, las entradas se acumulan y la versión se sube una vez, al cerrar.
5. `pnpm typecheck && pnpm humo`: lo que falle es el código a adaptar. Si es poco (un
   manejador 501 que sobra o falta), ahora; si no, tareas en la cola.
6. Si el cambio afecta a una tarea que ya tiene plan, actualiza `plans/<ID>.md`.

Todo código que escribas aquí o en cualquier modo sigue `estilo.md`.

## §E · Auditoría

- **De estructura**: compara el repo con `estructura.md` y las puertas de `flujo-y-puertas.md`.
- **De completitud** ("¿está listo para implementar?"): recorre `completitud.md` contra los
  fragmentos del contrato, sin leer el bundle.

Entrega un informe en `docs/`: veredicto corto, lo que está bien (tabla), problemas con
archivo y línea, estado de cada puerta y plan en orden. Ofrece convertir el plan en la cola.

## §P · Planificar → `/dbc:plan`

Un modelo planifica (`plans/<ID>.md`, plantilla en `plantillas.md` §17) y otro implementa
leyendo solo el plan. Los pasos viven en `${CLAUDE_PLUGIN_ROOT}/skills/plan/SKILL.md`. Úsalo cuando
el usuario trabaja con dos modelos o una tarea es delicada; si no, `/dbc:next` basta.

## §I · Integraciones y CLI (pregunta siempre, no supongas)

Añadir una integración que no hace falta es coste y superficie de fallo; olvidar una que sí
obliga a rediseñar el contrato. Se **pregunta**, y lo descartado también se anota.

1. **Pregunta** con `AskUserQuestion`:
   - *¿Qué necesita este sistema?* (múltiple): correo · pagos · archivos · CLI.
   - *¿En qué fase está?*: validación ($0) · primer cliente que paga · crecimiento. (El
     hosting siempre se decide; la fase dice cuál.)
   - Si eligió pagos: *¿qué vende y a quién?* — SaaS a todo el mundo · e-commerce físico ·
     Latinoamérica con medios locales · también cripto.
   - Lo que el contrato ya da por supuesto **también se confirma** (proveedor de pagos, de
     voz…), con su ADR: un proveedor heredado sin ADR es una decisión que nadie tomó.
2. **Lee solo la sección que toca** de `integraciones.md` (o `cli.md`) y recomienda **una**
   opción por tipo, con porqué y coste. Si la verificación de precios tiene más de 3 meses, o
   el proveedor no está en la referencia, compruébalo con WebSearch antes de recomendar.
3. **Registra** (`integraciones.md` § Cómo registrar): ADR, `docs/architecture.md`,
   `.env.example`, operaciones en el contrato (§D) y tareas en la cola. Lo descartado, en "No usa".
4. **No implementes nada aquí**: entra como tareas, con su contrato ya escrito.

## Economía de tokens (aplica siempre)

- `CLAUDE.md` raíz corto (≈40 líneas): reglas y punteros. Se carga en cada sesión.
- `.claude/settings.json` deniega leer y editar lo generado.
- Nada de diagramas, PDFs ni lockfiles salvo que la tarea lo pida. `grep -n` antes de abrir.
- Referencias: solo la sección que toca, nunca el archivo entero.
- Una cola que miente obliga a releer todo: si `PROGRESO.md` está desactualizado, arréglalo antes de seguir.
- Entre tareas, `/clear` mejor que `/compact`: el estado vive en la cola y en los planes.
