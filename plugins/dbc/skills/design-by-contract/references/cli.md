# CLI contra el backend

El CLI es **un consumidor más del contrato**, al mismo nivel que el frontend: usa los tipos
generados, no inventa rutas y, si necesita algo que el contrato no ofrece, se cambia
primero el contrato (§D).

## Dónde vive

```
cli/
├── package.json        "bin": { "<proyecto>": "./dist/index.js" }, depende de @<proyecto>/contrato
├── src/
│   ├── index.ts        registro de comandos
│   ├── api.ts          cliente openapi-fetch + token + manejo de errores RFC 9457
│   ├── credenciales.ts guardar/leer token
│   └── comandos/<recurso>.ts   un archivo por tag del contrato
├── tsconfig.json
└── PROGRESO.md
```

Añade `'cli'` a `pnpm-workspace.yaml`.

Dependencias mínimas: `commander` y `openapi-fetch`. Para compilar a un solo archivo:
`tsup`. No añadas librerías de colores, spinners ni prompts interactivos hasta que hagan
falta de verdad.

## El contrato debe declarar cómo se autentica el CLI

El navegador usa la sesión (cookie); un CLI no puede. Añade un esquema al contrato antes de
escribir el CLI:

- **MVP:** tokens personales de API (`Authorization: Bearer <token>`) que el usuario crea y
  revoca desde el panel. Requiere en el contrato: `POST /v1/tokens`, `GET /v1/tokens`,
  `DELETE /v1/tokens/{id}`, y el esquema `bearerAuth` en las operaciones que use el CLI.
- **Después:** *device code flow* (`cli login` abre el navegador) si los usuarios no
  técnicos van a usar el CLI.

## Comandos

Forma: `<proyecto> <recurso> <acción> [args] [--opciones]`, donde el recurso es el tag del
contrato y cada acción corresponde a **un** `operationId`.

```
miapp login                      guarda el token (lo pide sin mostrarlo)
miapp logout
miapp pedidos listar [--estado activo] [--json]
miapp pedidos ver <id>
miapp pedidos crear --archivo pedido.json     (o --campo valor …)
```

Opciones globales: `--json` (salida para scripts), `--api-url` (por defecto producción;
variable `MIAPP_API_URL`), `--help` en todos los niveles.

## Comportamiento que no se negocia

- **Token:** en `~/.config/<proyecto>/credenciales.json` con permisos `0600`. La variable
  `<PROYECTO>_TOKEN` tiene prioridad (CI y scripts). El token nunca se imprime ni se escribe
  en logs.
- **Salida:** tabla legible por defecto; `--json` imprime solo JSON válido en stdout. Los
  mensajes para personas van a stderr, así `--json` se puede encadenar con `jq`.
- **Errores:** muestra `title`, `detail` y cada `errores_validacion` del cuerpo RFC 9457.
- **Códigos de salida:** `0` bien · `1` uso incorrecto · `2` error 4xx de la API ·
  `3` error 5xx o de red.
- **Escrituras peligrosas** (borrar, cancelar, cobrar): piden confirmación salvo con `--si`.
- **Idempotencia:** las operaciones que el contrato marca con `Idempotency-Key` generan una
  clave por invocación (`crypto.randomUUID()`), así un reintento manual no duplica.

## Esqueleto

```ts
// src/api.ts
import createClient from 'openapi-fetch';
import type { paths } from '@<proyecto>/contrato';
import { leerToken } from './credenciales.js';

// index.ts copia --api-url a MIAPP_API_URL en un hook preAction del programa raíz.
export const api = () =>
  createClient<paths>({
    baseUrl: process.env['MIAPP_API_URL'] ?? 'https://api.miapp.com',
    headers: { Authorization: `Bearer ${leerToken()}` },
  });

export function fallar(error: { title?: string; detail?: string; status?: number;
  errores_validacion?: { campo: string; mensaje: string }[] }): never {
  console.error(`✗ ${error.title ?? 'Error'}${error.detail ? ` — ${error.detail}` : ''}`);
  for (const e of error.errores_validacion ?? []) console.error(`  ${e.campo}: ${e.mensaje}`);
  process.exit(error.status && error.status < 500 ? 2 : 3);
}
```

```ts
// src/comandos/pedidos.ts
import { Command } from 'commander';
import { api, fallar } from '../api.js';

export const comandos = [
  { operationId: 'listarPedidos', registrar: (padre: Command) =>
      padre.command('listar').option('--estado <estado>').option('--json')
        .action(async (o) => {
          const { data, error } = await api().GET('/v1/pedidos', { params: { query: { estado: o.estado } } });
          if (error) fallar(error);
          o.json ? console.log(JSON.stringify(data)) : console.table(data.datos);
        }) },
];
```

Como `openapi-fetch` tipa ruta, método y parámetros desde `paths`, **si el contrato cambia
y el CLI no se adapta, deja de compilar** (puerta 4).

## "CLI completo": comprobación de cobertura

Si el objetivo es que el CLI cubra toda la API pública, aplica el mismo freno que el backend:
cada comando declara su `operationId`, y una prueba compara esa lista con las operaciones
del contrato (excluyendo los tags `webhooks` e `interno`). Si falta alguna, la prueba lo dice;
si un comando apunta a un `operationId` que no existe, también.

```ts
// cli/src/cobertura.test.ts  (node --test)
import assert from 'node:assert/strict';
// operationIds del contrato: leer contracts/dist/openapi.bundled.yml con `yaml` y filtrar por tag
// operationIds del CLI: reunir `comandos[].operationId` de src/comandos/*
assert.deepEqual(faltan, [], `Operaciones sin comando: ${faltan.join(', ')}`);
assert.deepEqual(sobran, [], `Comandos sin operación en el contrato: ${sobran.join(', ')}`);
```

Si el CLI solo cubre una parte de la API a propósito, lista las exclusiones en
`cli/PROGRESO.md` en vez de desactivar la prueba.

## Prueba de humo

Contra el mock: `pnpm mock` y luego
`MIAPP_API_URL=http://localhost:4010 MIAPP_TOKEN=x node cli/dist/index.js pedidos listar --json | jq .`
debe salir con código 0 y JSON válido.

## Distribución

- Uso propio o del equipo: `pnpm --filter cli build && pnpm link --global`.
- Público: publicar en npm (`npx <proyecto>`). Versiona el CLI con SemVer y declara en su
  `README` qué versión mínima del contrato necesita.
