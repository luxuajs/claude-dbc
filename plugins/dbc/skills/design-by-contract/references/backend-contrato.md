# El backend carga el contrato

La idea: las rutas **no se declaran en el código**. Se leen del bundle, se registran con los
esquemas del contrato (así la validación de entrada *es* el contrato), y el proceso **no
arranca** si hay un manejador sin operación o una operación sin manejador.

## Fastify (recomendado)

Paquete `backend/packages/contrato` (o `backend/src/contrato/` en pequeño). Dependencias:
`fastify`, `ajv-formats`, `yaml`.

### `cargar.ts` — del bundle a operaciones

```ts
import { readFileSync } from 'node:fs';
import { parse } from 'yaml';

const METODOS = ['get', 'post', 'put', 'patch', 'delete'] as const;
type Metodo = (typeof METODOS)[number];

export interface Operacion {
  operationId: string;
  metodo: Metodo;
  ruta: string;            // formato Fastify: /v1/x/:id
  servicio: string;        // primer tag
  esquema: { body?: unknown; params?: unknown; querystring?: unknown; headers?: unknown; response?: Record<string, unknown> };
}

interface Parametro { name: string; in: 'path' | 'query' | 'header' | 'cookie'; required?: boolean; schema?: object }

function agrupar(params: Parametro[], donde: Parametro['in']) {
  const propios = params.filter((p) => p.in === donde);
  if (propios.length === 0) return undefined;
  const properties: Record<string, unknown> = {};
  const required: string[] = [];
  for (const p of propios) {
    const clave = donde === 'header' ? p.name.toLowerCase() : p.name; // Node baja las cabeceras
    properties[clave] = p.schema ?? { type: 'string' };
    if (p.required) required.push(clave);
  }
  return { type: 'object', properties, ...(required.length ? { required } : {}),
           ...(donde === 'header' ? { additionalProperties: true } : {}) };
}

export function cargarContrato(ruta: string) {
  // maxAliasCount -1: el bundle desreferenciado repite esquemas como anclas YAML.
  const doc = parse(readFileSync(ruta, 'utf8'), { maxAliasCount: -1 }) as any;
  const operaciones = new Map<string, Operacion>();

  for (const [rutaContrato, item] of Object.entries<any>(doc.paths ?? {})) {
    for (const metodo of METODOS) {
      const op = item[metodo];
      if (!op) continue;
      if (!op.operationId) throw new Error(`Sin operationId: ${metodo.toUpperCase()} ${rutaContrato}`);
      if (operaciones.has(op.operationId)) throw new Error(`operationId duplicado: ${op.operationId}`);
      if (!op.tags?.length) throw new Error(`"${op.operationId}" sin tags: el primero nombra el servicio`);

      const params: Parametro[] = [...(item.parameters ?? []), ...(op.parameters ?? [])];
      const response: Record<string, unknown> = {};
      for (const [codigo, r] of Object.entries<any>(op.responses ?? {})) {
        const s = r?.content?.['application/json']?.schema ?? r?.content?.['application/problem+json']?.schema;
        if (s) response[codigo] = s;
      }
      operaciones.set(op.operationId, {
        operationId: op.operationId,
        metodo,
        ruta: rutaContrato.replace(/\{([^}]+)\}/g, ':$1'),
        servicio: op.tags[0],
        esquema: {
          body: op.requestBody?.content?.['application/json']?.schema,
          params: agrupar(params, 'path'),
          querystring: agrupar(params, 'query'),
          headers: agrupar(params, 'header'),
          response: Object.keys(response).length ? response : undefined,
        },
      });
    }
  }
  return {
    version: doc.info?.version ?? '0.0.0',
    porServicio: (s: string) => [...operaciones.values()].filter((o) => o.servicio === s),
  };
}
```

### `servicio.ts` — montar y frenar la deriva

```ts
import Fastify, { type RouteHandlerMethod } from 'fastify';
import addFormats from 'ajv-formats';
import { cargarContrato } from './cargar.js';

export type Manejadores = Record<string, RouteHandlerMethod>;

export async function crearServicio(servicio: string, manejadores: Manejadores, rutaBundle: string) {
  const contrato = cargarContrato(rutaBundle);
  const operaciones = contrato.porServicio(servicio);

  // El freno: si código y contrato no coinciden, el proceso no arranca.
  const declaradas = new Set(operaciones.map((o) => o.operationId));
  const faltan = [...declaradas].filter((id) => !(id in manejadores));
  const sobran = Object.keys(manejadores).filter((id) => !declaradas.has(id));
  if (faltan.length || sobran.length) {
    throw new Error(
      `"${servicio}" no coincide con el contrato.` +
      (faltan.length ? `\n  Sin manejador: ${faltan.join(', ')}` : '') +
      (sobran.length ? `\n  Fuera del contrato: ${sobran.join(', ')} — añádelas al contrato (CAMBIOS.md), no quites esta comprobación.` : ''),
    );
  }

  const app = Fastify({
    // OpenAPI 3.1 trae `examples`, `summary`… que ajv estricto rechaza: son documentación.
    ajv: { customOptions: { strict: false, allErrors: true, coerceTypes: 'array', useDefaults: true },
           plugins: [addFormats as never] },
  });

  app.setErrorHandler((error: any, req, res) => {
    if (error.validation) {
      return res.status(422).type('application/problem+json').send({
        type: '/errores/validacion', title: 'La petición no cumple el contrato', status: 422,
        instance: req.url, request_id: String(req.id),
        errores_validacion: error.validation.map((v: any) => ({
          campo: v.instancePath || `/${v.params?.missingProperty ?? ''}`, mensaje: v.message ?? 'valor inválido' })),
      });
    }
    // Errores con statusCode son nuestros (4xx, 501): se muestran. Los demás, sin detalles.
    const status = error.statusCode ?? 500;
    return res.status(status).type('application/problem+json').send({
      type: error.tipo ?? '/errores/interno', title: error.statusCode ? error.message : 'Error interno',
      status, instance: req.url, request_id: String(req.id) });
  });

  app.setNotFoundHandler((req, res) => res.status(404).type('application/problem+json')
    .send({ type: '/errores/ruta-no-encontrada', title: 'Esa ruta no está en el contrato', status: 404, instance: req.url }));

  // Sonda de la plataforma; fuera del contrato a propósito.
  app.get('/salud', async () => ({ servicio, contrato: contrato.version, operaciones: operaciones.length }));

  for (const op of operaciones) {
    app.route({ method: op.metodo.toUpperCase() as 'GET', url: op.ruta, schema: op.esquema as never,
                handler: manejadores[op.operationId]! });
  }
  return app;
}
```

### Manejadores: uno por `operationId`, 501 hasta implementarlos

```ts
const sinImplementar = (op: string) =>
  Object.assign(new Error(`<servicio>.${op} sin implementar`), { statusCode: 501, tipo: '/errores/sin-implementar' });

export const manejadores: Manejadores = {
  listarPedidos: async () => { throw sinImplementar('listarPedidos'); },
  crearPedido:   async () => { throw sinImplementar('crearPedido'); },
};
```

Genera esta lista desde el contrato (`grep -n operationId contracts/<servicio>/paths.yml`)
cada vez que el contrato añade operaciones: el servicio no arrancará hasta que estén todas.

### Tipar los manejadores con el contrato

```ts
import type { operations } from '@<proyecto>/contrato';
type Cuerpo<Op extends keyof operations> =
  operations[Op] extends { requestBody?: { content: { 'application/json': infer C } } } ? C : never;
```

## Prueba de humo (`tools/scripts/humo.ts`)

Debe demostrar, con `node:assert`, que:
1. cada servicio arranca y responde `/salud`;
2. una petición sin un campo obligatorio devuelve 422 `application/problem+json` con `errores_validacion`;
3. una ruta que el contrato no declara devuelve 404;
4. `crearServicio` con un manejador extra (`rutaFantasma`) **lanza** al arrancar.

Usa `app.inject()` de Fastify: no hace falta levantar puertos.

## Express

Usa `express-openapi-validator` apuntando al bundle (`apiSpec`, `validateRequests: true`,
`validateResponses: true` en desarrollo). Para el freno de deriva, recorre `doc.paths`,
reúne los `operationId` y compáralos con las claves de tu objeto de manejadores antes de
`app.listen()`, igual que en Fastify. Si el proyecto existente ya usa Express, quédate en
Express: cambiar de framework no es parte de adoptar la metodología.

## Frontend

```ts
import createClient from 'openapi-fetch';
import type { paths } from '@<proyecto>/contrato';

export const api = createClient<paths>({ baseUrl: process.env.NEXT_PUBLIC_API_URL });
// api.GET('/v1/pedidos/{id}', { params: { path: { id } } })  ← ruta, método y params tipados
```

En desarrollo, `NEXT_PUBLIC_API_URL=http://localhost:4010` (el mock de Prism): el frontend
avanza sin esperar al backend. Con varios servicios, una URL base por servicio, elegida por el
primer `tag` de la operación.

El frontend es **solo cliente** del contrato: sin Server Actions ni rutas API de Next que
hagan de backend. Si Next hiciera de backend, habría una API que el contrato no declara.
