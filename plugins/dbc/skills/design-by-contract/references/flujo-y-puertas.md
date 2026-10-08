# Flujo bidireccional y puertas

## El flujo

```
         ┌──────────────── IDEA / REQUISITO ────────────────┐
         ▼                                                  │
 1. Cambiar contracts/ (+ CAMBIOS.md, + ADR si es decisión)│
         ▼                                                  │
 2. pnpm contrato  → lint + bundle + tipos                  │
         ▼                                                  │
 3. Frontend contra el mock · Backend implementa            │
         ▼                                                  │
 4. ¿La implementación puede cumplir el contrato?           │
       ├── sí → 5. PR: CI verifica las puertas → merge      │
       └── no → SE DETIENE EL CÓDIGO ───────────────────────┘
                el primer commit de la corrección va al contrato
```

"Bidireccional" significa que la información vuelve del código al contrato, **pero siempre
pasando por el contrato**: el código nunca adelanta al contrato aunque sea el que descubre
el problema.

Ejemplos de vuelta atrás legítima:
- El proveedor de pagos devuelve el importe en centavos enteros y el contrato decía decimal → se cambia el contrato y se documenta el motivo.
- Una pantalla necesita un campo que la respuesta no trae → tarea de contrato antes de tocar el frontend.
- Un listado sin paginar se vuelve lento → se añade paginación en el contrato (cambio incompatible si quita la forma anterior).

## Las puertas

| # | Puerta | Herramienta | Detecta | Tamaño |
|---|---|---|---|---|
| 1 | Lint | `redocly lint` | Contrato mal formado, refs rotas, sin `operationId` | todos |
| 2 | Frescura | regenerar + `git diff --exit-code contracts/generated contracts/dist` | Contrato editado sin regenerar | mediano+ |
| 3 | Compatibilidad | `oasdiff breaking base.yml rama.yml --fail-on ERR` | Ruptura no intencionada | mediano+ |
| 4 | Tipos | `pnpm typecheck` | Código que no se adaptó al contrato | todos |
| 5 | Deriva | `pnpm humo` (cargador + freno al arrancar) | Manejadores que faltan o sobran; entrada que no valida | todos |
| 6 | Bitácora | si cambia `contracts/**/*.yml`, exigir `CAMBIOS.md` | Cambio sin explicar | mediano+ |
| 7 | Respuestas | `prism proxy <bundle> <url> --errors` en integración | Respuesta distinta a la prometida | cuando haya lógica |

Nota sobre la puerta 7: si el backend registra los esquemas de respuesta en Fastify, éste
los usa para **serializar** (descarta campos extra y convierte tipos en silencio). Eso
protege, pero no avisa. Prism en modo proxy sí avisa.

Recomendación: **versionar `contracts/dist/` y `contracts/generated/`**. Hace posible la
puerta 3 sin pasos extra y da un diff legible del contrato en cada PR.

## Workflow de CI

`.github/workflows/contrato.yml`:

```yaml
name: contrato
on: pull_request
jobs:
  puertas:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with: { fetch-depth: 0 }
      - uses: pnpm/action-setup@v4
      - uses: actions/setup-node@v4
        with: { node-version: 22, cache: pnpm }
      - run: pnpm install --frozen-lockfile
      - run: pnpm contrato                                       # 1 lint + bundle + tipos
      - run: git diff --exit-code contracts/generated contracts/dist   # 2 frescura
      - name: Cambios incompatibles                              # 3
        if: ${{ !contains(github.event.pull_request.labels.*.name, 'contrato-incompatible') }}
        run: |
          git show origin/${{ github.base_ref }}:contracts/dist/openapi.bundled.yml > base.yml || exit 0
          docker run --rm -v "$PWD:/w" -w /w tufin/oasdiff \
            breaking base.yml contracts/dist/openapi.bundled.yml --fail-on ERR
      - name: CAMBIOS.md acompaña al contrato                    # 6
        run: |
          base=origin/${{ github.base_ref }}
          if git diff --name-only $base... | grep -qE '^contracts/.*\.ya?ml$'; then
            git diff --name-only $base... | grep -q '^contracts/CAMBIOS.md$' \
              || { echo "El contrato cambió sin entrada en CAMBIOS.md"; exit 1; }
          fi
      - run: pnpm typecheck                                      # 4
      - run: pnpm humo                                           # 5
```

En un proyecto pequeño basta con los pasos de install, `pnpm contrato`, `typecheck` y `humo`.

La etiqueta `contrato-incompatible` en el PR salta la puerta 3. Solo se pone junto con un ADR
que explique la ruptura: así es una decisión y no un accidente.

## Qué cuenta como cambio incompatible

Quitar un endpoint o un campo de respuesta, hacer obligatorio un campo que era opcional,
añadir un campo obligatorio a una petición, estrechar un `enum`, cambiar el tipo de un campo,
cambiar el significado de un valor aunque el tipo no cambie, cambiar códigos de estado.

Mientras solo hay clientes internos (tu frontend, tus servicios), un cambio incompatible se
aplica y se despliega todo junto. Con un integrador externo eso deja de valer: se abre `/v2`.

## Versionado

`info.version` con SemVer: incompatible → mayor; añade algo compatible → menor; aclaraciones
o ejemplos → parche. Cada versión tiene su entrada en `CAMBIOS.md`.
