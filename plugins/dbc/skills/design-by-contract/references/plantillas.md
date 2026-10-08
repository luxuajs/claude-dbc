# Plantillas

Sustituye `<proyecto>`, `<servicio>`, `<fecha>`. Adapta, no copies a ciegas: si el proyecto
no tiene varios servicios, quita lo que hable de ellos.

## Índice
1. CLAUDE.md raíz · 2. PROGRESO.md raíz (cola) · 3. PROGRESO.md de carpeta ·
4. PROGRESO.md de servicio · 5. CAMBIOS.md · 6. ADR · 7. architecture.md ·
8. .claude/settings.json · 9. package.json (scripts) · 10. pnpm-workspace.yaml ·
11. .gitignore · 12. redocly.yaml · 13. openapi.yml raíz · 14. common/errores.yml ·
15. Runbook · 16. Plantilla de PR · 17. Plan de tarea (`plans/<ID>.md`)

---

## 1. `CLAUDE.md` raíz

```markdown
# <proyecto> — reglas para el agente

## Regla 1: el contrato va primero

`contracts/openapi.yml` es el sistema. No se escribe código que el contrato no declare.
Si la implementación no puede cumplir el contrato: **se detiene el código**, se cambia el
contrato con su entrada en `contracts/CAMBIOS.md`, y después se sigue.

## Al empezar una sesión

1. Lee `PROGRESO.md` (raíz). Trabaja **solo** en la tarea *En curso*, o en la primera
   pendiente si no hay ninguna. Una tarea por sesión.
2. Lee el `PROGRESO.md` de la carpeta (o servicio) de esa tarea.
3. Lee **solo** lo que la tarea indica en "Leer". Busca con `grep -n` antes de abrir
   archivos enteros.

**Si la tarea tiene plan (`plans/<ID>.md`), el plan manda:** lee solo el plan y los archivos
que nombra; ni la cola, ni el contrato, ni otros `PROGRESO.md`. Si algo choca con el contrato,
**para** y escríbelo en su sección *Bloqueos*. Los planes se escriben con `plans/_template.md`.
<!-- Este párrafo solo cuando el proyecto usa plans/ (SKILL.md §P). -->

## Estilo de código

KISS primero; SOLID, DRY y patrones solo donde hay variación real (un adaptador por
proveedor, no interfaces "por si acaso"). Comentarios en español, breves: qué hace y por qué.
Seguridad según OWASP Top 10:2025: tenant y rol desde la sesión, consultas parametrizadas,
firmas verificadas, fallar cerrado, nada sensible en logs.

## Stack

Las tecnologías y cómo se usa cada una están en `docs/architecture.md` § Stack. Léelo antes
de añadir una dependencia o de elegir cómo resolver algo; si cambias el stack, actualízalo.

## No leer salvo que se pida

- `contracts/dist/` y `contracts/generated/` — generados y enormes. Para la forma de una
  operación, lee su fragmento en `contracts/<servicio>/`.
- `docs/diagramas/`, `docs/negocio/`, PDFs, lockfiles, `node_modules/`.

## Cambiar el contrato

1. Editar el fragmento: `contracts/<servicio>/{paths,schemas}.yml` o `contracts/common/`.
2. `pnpm contrato` (lint + bundle + tipos).
3. Entrada en `contracts/CAMBIOS.md`: qué cambió y **por qué**. Subir `info.version`.
4. `pnpm typecheck && pnpm humo` — lo que falle es el código que hay que adaptar.

Nunca se editan a mano `contracts/dist/` ni `contracts/generated/`.

## Al terminar una tarea

1. Verificar con el comando de "Hecho cuando".
2. En `PROGRESO.md` (raíz): moverla a *Hecho* en una línea con fecha; indicar la siguiente.
3. Actualizar el `PROGRESO.md` de la carpeta/servicio si cambió su estado.

## Comandos

pnpm contrato · pnpm typecheck · pnpm humo · pnpm mock (:4010) · pnpm dev
```

## 2. `PROGRESO.md` raíz (la cola)

```markdown
# Progreso — cola de trabajo

Única lista de **qué falta y en qué orden**. Los `PROGRESO.md` de cada carpeta dicen
**cómo está** esa carpeta; no llevan pendientes propios.

## Reglas

1. **Una tarea por sesión.** Si no cabe, se parte en `T07a`, `T07b`… antes de empezar.
2. **Primero el tronco de contrato.** Mientras tenga tareas pendientes, no existen tareas de
   backend ni de frontend: las crea la tarea "derivar" cuando el contrato está cerrado.
   Después, **como mucho una tarea en curso por carril**.
3. Cada tarea dice **qué leer** (y nada más) y **cuándo está hecha** (un comando que pasa).
4. Al terminar: a *Hecho* con fecha en una línea; actualizar el `PROGRESO.md` de la carpeta.
5. Una tarea que choca con el contrato **no sigue**: antes se crea una tarea de contrato y
   la original queda *bloqueada por* ella.

## En curso

_Ninguna._ La siguiente es **C01**.
<!-- Con carriles: una línea por carril ("Backend: … · Frontend: …"). -->

## Bloqueadas

_Ninguna._

## Pendiente (en orden)

### Tronco — contrato completo, antes que todo
- [ ] **C01 · <título>** — `contracts/<servicio>/`
  - Leer: <archivos concretos>
  - Hacer: <qué, en una o dos líneas>
  - Hecho cuando: <comando que pasa o comprobación objetiva>
- [ ] **Cnn · Cerrar el contrato** — `references/completitud.md` § Cierre; versión + tag.
- [ ] **Cnn · Derivar las tareas de backend y frontend del contrato cerrado** — crea los dos
  carriles de abajo; cada `operationId` en exactamente una tarea de backend.

### Común — en cualquier momento (estructura, CI, docs)

### Carril backend (lo crea "Derivar")
<!-- Primero lo común (SQL, auth, cargador, idempotencia, adaptadores, integraciones);
     después una tarea por grupo pequeño de operationId de un mismo servicio. -->

### Carril frontend (lo crea "Derivar"; contra `pnpm mock`)
<!-- Primero la base (cliente tipado, sesión, errores); después una tarea por pantalla con
     sus operationId. Las operaciones sin pantalla (webhooks, internas) se listan con motivo. -->

### Final — antes de desplegar (validación de respuestas, despliegue, runbooks, topes de gasto)

## Hecho

- [x] <fecha> · <tarea en una línea>
```

Cuando *Hecho* pase de ~30 líneas, mueve las más antiguas a `docs/historial.md`: la cola
se lee en cada sesión y debe seguir siendo corta.

## 3. `PROGRESO.md` de carpeta

```markdown
# Progreso — <carpeta>

**Estado: <una frase>.**

## Qué funciona hoy
- <hecho verificable, con el comando que lo verifica>

## Lo siguiente
En la cola única: [`/PROGRESO.md`](../PROGRESO.md) — tareas <Txx–Tyy>.

## Decidido
- <decisión> — ver `docs/decisions/NNNN-*.md`.
```

## 4. `PROGRESO.md` de servicio

Créalo con la **primera tarea que añade lógica** al servicio, no antes.

```markdown
# Progreso — <servicio>

Esquema: `<servicio>` · Puerto: <n> · Pide por REST a: <servicios o —>

## Operaciones (operationId del contrato)
- [x] <operationId> — <fecha>
- [ ] <operationId>   ← devuelve 501 hasta que se implemente

## Rarezas de este servicio
- <lo que una sesión futura necesita saber y no está en el contrato>
```

La lista de operaciones sale de `grep -n "operationId" contracts/<servicio>/paths.yml`.

## 5. `contracts/CAMBIOS.md`

```markdown
# Bitácora del contrato

## La regla
El contrato manda. Cuando la implementación no puede cumplirlo, **se cambia el contrato,
no se ignora**, en el mismo PR y con su entrada aquí.

## Entradas

### <fecha> — v<x.y.z>, <título>
- **Qué cambió:** <operaciones, campos>
- **Por qué:** <por qué la implementación no podía cumplir lo anterior>
- **Compatibilidad:** compatible | incompatible (ADR NNNN)
```

## 6. ADR — `docs/decisions/NNNN-<titulo-en-kebab>.md`

```markdown
# NNNN — <decisión en una frase>

- Estado: propuesta | aceptada | reemplazada por NNNN
- Fecha: <fecha>

## Contexto
<qué problema o fuerza obliga a decidir>

## Decisión
<lo que se decidió>

## Consecuencias
<qué se gana, qué cuesta, qué queda más difícil>
```

Un ADR no se edita cuando cambia la decisión: se escribe otro que lo reemplaza.
El ADR 0001 de todo proyecto es "Usamos design by contract con OpenAPI".

## 7. `docs/architecture.md`

```markdown
# Arquitectura — <proyecto>

## En una frase

## Stack
| Capa | Tecnología | Para qué se usa aquí | Cómo se usa (convención del proyecto) | ADR |
|---|---|---|---|---|
| Contrato | OpenAPI 3.1 + Redocly | Fuente de verdad de la API | Fragmentos por dominio; `pnpm contrato` regenera bundle y tipos | 0001 |
| Backend | Node 22 + Fastify | Servicios HTTP | Rutas cargadas del contrato, nunca declaradas a mano; un manejador por `operationId` | 0001 |
| Base de datos | PostgreSQL + Prisma | Persistencia | Un esquema y un rol por servicio; migraciones con `prisma migrate` | NNNN |
| Frontend | Next.js + openapi-fetch | Panel / web | Cliente tipado desde el contrato; mock de Prism en desarrollo | NNNN |
| Monorepo | pnpm workspaces + TypeScript | Orquestación | Scripts en `package.json` raíz | — |
| … | | | | |

Sin versiones exactas: viven en `package.json` y el lockfile, y repetirlas aquí hace que
este documento mienta en la siguiente actualización. Como mucho, la versión mayor
cuando importa (Next 15, Node 22).

## Piezas (tabla: pieza · qué hace · dónde se despliega · coste)
## Quién habla con quién (diagrama C4 de contenedores o lista)
## Dónde vive cada dato
## Reglas que atraviesan todo (las del info.description del contrato, resumidas)
## Servicios externos (tabla: tipo · proveedor · plan · coste · ADR · cuándo revisar)
## No usa (integraciones descartadas a propósito: correo, pagos, archivos, CLI…)
## Diagramas → docs/diagramas/
```

1–3 páginas. Si crece más, sobra detalle que pertenece al contrato o a un ADR.

**El Stack se mantiene vivo:** añadir, quitar o sustituir una tecnología con papel propio
(framework, ORM, proveedor, librería que define cómo se hace algo) actualiza su fila en la
misma tarea, y si la decisión es difícil de revertir, lleva ADR. Las utilidades menores
(una librería de fechas) no entran.

## 8. `.claude/settings.json`

```json
{
  "permissions": {
    "deny": [
      "Read(contracts/dist/**)",
      "Read(contracts/generated/**)",
      "Edit(contracts/dist/**)",
      "Edit(contracts/generated/**)"
    ]
  }
}
```

Si el proyecto ya tiene `settings.json`, añade estas reglas a su `deny`, no lo reemplaces.

## 9. `package.json` raíz (scripts)

```json
{
  "private": true,
  "type": "module",
  "engines": { "node": ">=22" },
  "scripts": {
    "contrato:lint": "redocly lint --config contracts/redocly.yaml",
    "contrato:bundle": "redocly bundle contracts/openapi.yml --dereferenced -o contracts/dist/openapi.bundled.yml",
    "contrato:tipos": "pnpm contrato:bundle && openapi-typescript contracts/dist/openapi.bundled.yml -o contracts/generated/contrato.d.ts",
    "contrato": "pnpm contrato:lint && pnpm contrato:tipos",
    "dev": "pnpm --parallel --filter './backend/**' dev",
    "typecheck": "pnpm -r typecheck",
    "test": "pnpm -r test",
    "humo": "tsx tools/scripts/humo.ts",
    "mock": "prism mock contracts/dist/openapi.bundled.yml --port 4010"
  },
  "devDependencies": {
    "@redocly/cli": "latest",
    "openapi-typescript": "latest",
    "@stoplight/prism-cli": "latest",
    "typescript": "latest",
    "tsx": "latest"
  }
}
```

Fija las versiones al instalar (`pnpm add -D`), no dejes `latest` en el repo.

## 10. `pnpm-workspace.yaml`

```yaml
packages:
  - 'contracts'
  - 'backend/packages/*'
  - 'backend/services/*'
  - 'frontend/*'
  - 'cli'            # solo si hay CLI
```

En pequeño: `'contracts'`, `'backend'`, `'frontend/*'`.

## 11. `.gitignore`

```
node_modules/
dist/
!contracts/dist/
.env
.env.*
!.env.example
.next/
*.tsbuildinfo
```

`contracts/dist/` se versiona a propósito (puerta 3 y diff legible del contrato).

## 12. `contracts/redocly.yaml`

```yaml
# Las reglas que se apagan se apagan aquí, con su motivo escrito.
extends:
  - recommended
apis:
  main:
    root: ./openapi.yml
rules:
  operation-operationId: error
  operation-summary: error
  operation-description: warn
  no-ambiguous-paths: error
  no-unresolved-refs: error
  security-defined: error
  # no-server-example.com: off   # si declaras servidores localhost para desarrollo
```

## 13. `contracts/openapi.yml` raíz

```yaml
openapi: 3.1.0
info:
  title: <proyecto> — API
  version: 0.1.0
  description: |
    # El contrato
    Este documento **es** el sistema. Backend y frontend se construyen contra él.

    ## Regla bidireccional
    Cuando la implementación no puede cumplirlo, se cambia el contrato (con su entrada
    en CAMBIOS.md), nunca se ignora.

    ## Reglas que atraviesan todo el contrato
    - Errores RFC 9457 (`application/problem+json`), iguales en todo el sistema.
    - Toda operación tiene `operationId`; el primer `tag` nombra el servicio dueño.
    - `Idempotency-Key` en toda escritura del cliente que cueste dinero o cree algo que no
      debe duplicarse. Los webhooks no la llevan (ningún proveedor la manda): declaran su
      clave natural con `x-clave-idempotencia: 'cuerpo:/id'` y, si se traducen a un dialecto
      neutral, su `x-adaptador`; el cargador verifica la firma en crudo y traduce antes de validar.
    - Roles con `x-roles: [<rol>]` en la operación (403 al resto); recurso de otro tenant → 404.
    - Dinero como texto decimal + moneda, nunca número.
    - La identidad del usuario/tenant sale de la sesión, nunca de un parámetro.
servers:
  - url: http://localhost:3001
    description: Desarrollo
tags:
  - name: <servicio>
    description: <de qué es dueño>
security:
  - sesion: []
paths:
  /v1/<recurso>:
    $ref: './<servicio>/paths.yml#/<Recurso>'     # en pequeño: la operación va aquí mismo
components:
  securitySchemes:
    sesion:
      $ref: './common/comunes.yml#/seguridad/sesion'
```

## 14. `contracts/common/errores.yml`

```yaml
Problema:
  type: object
  description: Cuerpo de toda respuesta de error (RFC 9457). El cliente decide por `type`.
  required: [type, title, status]
  properties:
    type: { type: string, format: uri-reference }
    title: { type: string }
    status: { type: integer, minimum: 400, maximum: 599 }
    detail: { type: string }
    instance: { type: string, format: uri-reference }
    request_id: { type: string }
    errores_validacion:
      type: array
      description: Solo en 422. Un elemento por campo que no cumple el contrato.
      items:
        type: object
        required: [campo, mensaje]
        properties:
          campo: { type: string }
          mensaje: { type: string }

respuestas:
  NoAutorizado:
    description: Falta la sesión o no es válida.
    content:
      application/problem+json:
        schema: { $ref: '#/Problema' }
  Prohibido:
    description: La sesión no tiene permiso.
    content:
      application/problem+json:
        schema: { $ref: '#/Problema' }
  NoEncontrado:
    description: El recurso no existe.
    content:
      application/problem+json:
        schema: { $ref: '#/Problema' }
  NoProcesable:
    description: La petición no cumple el contrato.
    content:
      application/problem+json:
        schema: { $ref: '#/Problema' }
```

## 15. Runbook — `docs/runbooks/<accion>.md`

```markdown
# <Qué hacer cuando…>

**Síntoma:** <cómo te enteras>
**Gravedad:** <qué se rompe para el cliente>

## Pasos
1. <comando exacto>
2. <qué mirar y qué valor es normal>

## Verificar que quedó resuelto
## Después: ¿hace falta un ADR o una tarea en la cola?
```

Runbooks típicos de un MVP: aplicar la base de datos, backup y restauración, rotar secretos,
topes de gasto en proveedores de pago por uso (IA, telefonía, SMS).

## 16. `.github/pull_request_template.md`

```markdown
## Qué cambia

## Contrato
- [ ] No toca el contrato
- [ ] Toca el contrato: entrada en `contracts/CAMBIOS.md` y `info.version` actualizada
- [ ] Cambio incompatible: ADR enlazado y etiqueta `contrato-incompatible`

## Cola
- [ ] Tarea movida a *Hecho* en `PROGRESO.md`
```

## 17. Plan de tarea — `plans/_template.md`

Solo en proyectos que usan `plans/` (SKILL.md §P). El que planifica copia la plantilla a
`plans/<ID>.md` y la rellena; el que implementa lee **solo** ese archivo. Una sección que no
aplique se deja con "No aplica", no se borra.

````markdown
# <ID> · <título de la tarea>

> Carril: backend | frontend · Contrato: v<x.y.z> · Plan escrito el <AAAA-MM-DD>
> Implementa: en UNA sesión. Lee solo este archivo y los archivos que nombra "Archivos".

## Objetivo
<2–3 líneas: qué existe al terminar que hoy no existe.>

## Archivos
- Crear: <rutas exactas>
- Modificar: <rutas exactas, y qué parte de cada una>
- No tocar: <rutas que parecen relacionadas y no lo son>

## Contrato aplicable
<Los bloques YAML de las operaciones y esquemas de esta tarea, COPIADOS de
`contracts/<servicio>/`. Si no toca operaciones, "No aplica".>

## Decisiones ya tomadas
- <decisión> — <por qué> (ADR NNNN). No se reabren aquí.

## Pasos
1. <paso concreto; en lo delicado, el código exacto: firmas, SQL, reglas de seguridad>

## Seguridad
- <riesgo OWASP Top 10:2025 que toca esta tarea> — <cómo queda cubierto, o "lo cubre el cargador/middleware X">

## Skills
- <skill y en qué paso>, o "Ninguna".

## No leer
- `contracts/dist/`, `contracts/generated/`, lockfiles, `node_modules/`, <otras>

## Verificación
```bash
<comandos exactos>
```
Salida esperada: <qué tiene que decir>.

## Cierre
- `PROGRESO.md` (raíz): <ID> a *Hecho* con fecha; en *En curso*, la siguiente de su carril.
- `<carpeta>/PROGRESO.md`: <qué marcar>.
- Commit: `<mensaje exacto>`

## Si algo choca con el contrato
PARAR. No cambiar el contrato ni improvisar. Escribir en *Bloqueos* qué no encaja y por qué,
hacer commit de lo hecho y terminar. El que planifica lo resuelve (§D) y actualiza el plan.

## Bloqueos
<Vacío. Solo se rellena si el implementador se detiene.>
````

Lo que hace barato un plan: **todo lo necesario va copiado dentro**. Un plan que dice
"lee el contrato" o "mira cómo está hecho X" devuelve al implementador al repo y se pierde
el ahorro.
