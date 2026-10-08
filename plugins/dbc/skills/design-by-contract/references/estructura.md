# Estructura estándar

## Árbol

`(S)` = mínimo para un proyecto pequeño. Lo demás entra cuando hace falta, como tarea de la cola.

```
mi-proyecto/
├── CLAUDE.md                     (S) reglas del repo para el agente (≈40 líneas)
├── README.md                     (S) qué es, cómo se arranca
├── PROGRESO.md                   (S) LA COLA: hecho / en curso / pendiente en orden
├── package.json                  (S) scripts: contrato, dev, typecheck, humo, mock
├── pnpm-workspace.yaml           (S) contracts, backend/*, frontend/*
├── tsconfig.base.json
├── .env.example
├── .gitignore
│
├── .claude/
│   ├── settings.json             (S) deniega leer/editar lo generado
│   └── skills/                        skills propias del proyecto, si las hay
│
├── .github/
│   ├── workflows/contrato.yml         las puertas de CI
│   └── pull_request_template.md       ¿toca el contrato? ¿CAMBIOS.md? ¿ADR?
│
├── plans/                             (si se planifica con un modelo e implementa con otro)
│   ├── _template.md                   plantilla de plan (plantillas.md §17)
│   └── <ID>.md                        un plan por tarea; el implementador lee solo este
│
├── docs/
│   ├── architecture.md           (S) STACK y CÓMO encajan las piezas (1–3 páginas)
│   ├── decisions/                (S) ADR: POR QUÉ se decidió algo
│   ├── runbooks/                      QUÉ HACER cuando algo falla
│   ├── diagramas/                     C4 (skill c4-arquitectura)
│   └── negocio/                       BPMN (skill bpmn-negocio), manuales, PDFs
│
├── contracts/                    (S) FUENTE DE VERDAD
│   ├── openapi.yml               (S) raíz: info, servers, tags, paths → $ref
│   ├── common/                   (S) errores.yml (RFC 9457), comunes.yml (paginación, cabeceras, seguridad)
│   ├── <dominio>/                     paths.yml + schemas.yml (a partir de tamaño mediano)
│   ├── redocly.yaml              (S) reglas de lint, con el motivo de cada regla apagada
│   ├── package.json                   paquete @<proyecto>/contrato que exporta los tipos
│   ├── generated/                     contrato.d.ts — GENERADO
│   ├── dist/                          openapi.bundled.yml — GENERADO
│   ├── CAMBIOS.md                (S) historial del contrato
│   └── PROGRESO.md               (S) estado del contrato
│
├── backend/
│   ├── packages/                      (mediano+) contrato (cargador), compartido, cliente-s2s
│   ├── services/<servicio>/           (mediano+) cada uno con su PROGRESO.md cuando tenga lógica
│   ├── src/                           (pequeño) un solo servicio
│   │   └── integraciones/             correo/, pagos/, archivos/ — un adaptador por proveedor
│   └── PROGRESO.md               (S)
│
├── frontend/
│   ├── web/                           consume @<proyecto>/contrato; usa el mock en desarrollo;
│   │                                  solo cliente: sin Server Actions ni rutas API como backend
│   └── PROGRESO.md               (S)
│
├── cli/                               (si se pidió) consumidor del contrato, ver cli.md
│   └── PROGRESO.md
│
├── database/
│   ├── migraciones/ o esquemas/       Prisma por servicio, o SQL
│   └── PROGRESO.md               (S)
│
└── tools/
    └── scripts/                       humo.ts y comprobaciones de CI
```

Dependencias permitidas (las flechas nunca apuntan hacia `contracts/` desde el código):

```
contracts/openapi.yml ──bundle──▶ contracts/dist ──▶ backend (cargador) y prism (mock)
                      ──tipos───▶ contracts/generated ──▶ backend y frontend
frontend y cli ──HTTP──▶ backend ──▶ database
                          backend ──adaptadores──▶ correo · pagos · archivos
```

El frontend y el CLI **no dependen de ningún paquete de `backend/`**: los tipos viven en `contracts/`.

## Tamaños

| | Pequeño (1 servicio, 1 dev) | Mediano (2–5 servicios) | Grande (muchos servicios o equipos) |
|---|---|---|---|
| Contrato | Un solo `openapi.yml` + `common/` | Raíz + fragmentos `<dominio>/` | Un `openapi.yml` por servicio + `common/`, versionados por separado |
| Backend | `backend/src/` | `backend/services/*` + `packages/*` | Igual, o un repo por servicio consumiendo contratos publicados |
| Tipos | `contracts/generated/` | Igual, como paquete del workspace | Paquete publicado por contrato con SemVer |
| Puertas de CI | 1, 4, 5 | 1–6 | 1–7 + pruebas consumidor-proveedor |
| Versionado | `info.version` | + `CAMBIOS.md` | SemVer por contrato; `/v2` con integradores externos |
| Docs | `architecture.md` + 2–3 ADR | + runbooks + C4 | + `architecture.md` por servicio |
| Eventos asíncronos | — | Webhooks en OpenAPI | Colas entre servicios → AsyncAPI en `contracts/eventos/` |
| Base de datos | Un esquema | Un esquema y un rol por servicio, sin FK cruzadas | Base propia por servicio |

Señales para subir de tamaño: aparece un segundo servicio desplegable → partir el contrato
por dominio. Aparecen equipos separados o integradores externos → un contrato por servicio
y `/v2` para romper compatibilidad.

## Qué responde cada documento

| Pregunta | Documento | No contiene |
|---|---|---|
| ¿QUÉ promete el sistema? | `contracts/openapi.yml` | Por qué, ni cómo se implementa |
| ¿QUÉ cambió en el contrato y cuándo? | `contracts/CAMBIOS.md` | Debates largos (enlaza al ADR) |
| ¿POR QUÉ se decidió así? | `docs/decisions/NNNN-*.md` | Detalles de implementación |
| ¿CON QUÉ está hecho y CÓMO se usa cada tecnología? ¿CÓMO encajan las piezas? | `docs/architecture.md` | Rutas o campos (contrato), versiones exactas (`package.json`) |
| ¿QUÉ HAGO si falla X? | `docs/runbooks/*.md` | Diseño |
| ¿QUÉ sigue y en qué orden? | `PROGRESO.md` raíz | Detalle de cada carpeta |
| ¿CÓMO está esta carpeta? | `<carpeta>/PROGRESO.md` | Pendientes propios (enlaza a la cola) |
| ¿CÓMO debe trabajar el agente? | `CLAUDE.md` | Documentación del producto |
| ¿CÓMO se hace ESTA tarea, paso a paso? | `plans/<ID>.md` | Otras tareas; lo que ya está en el contrato se copia, no se enlaza |

## Base de datos

- La base de datos **no** aparece en el contrato: es detalle de implementación.
- Con varios servicios: un esquema y un rol por servicio, con GRANT solo sobre lo suyo. Así
  la frontera entre servicios la impone la base de datos, no la disciplina.
- Sin claves foráneas entre esquemas de servicios distintos: la integridad se pide por una
  ruta interna del contrato (`/v1/internal/...`, firmada).
- Si el multi-tenant es posible en el futuro, `tenant_id` + RLS desde la primera migración.
