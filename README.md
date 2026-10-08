# claude-dbc

Marketplace del equipo con el plugin **`dbc`** (design by contract) para Claude Code.

## Instalar (una vez por persona)

```bash
claude plugin marketplace add luxuajs/claude-dbc
claude plugin install dbc@luxuajs
```

O dentro de Claude Code: `/plugin marketplace add luxuajs/claude-dbc` y `/plugin install dbc@luxuajs`.

Si el repositorio es privado, hace falta acceso de lectura a él con git (SSH o `gh auth login`).

## Actualizar

```bash
claude plugin marketplace update luxuajs
claude plugin update dbc@luxuajs
```

## Comandos

| Comando | Modelo | Para qué |
|---|---|---|
| `/dbc:plan <ID> [<ID>…]` | Opus | Escribe planes `plans/<ID>.md` con todo lo necesario copiado dentro |
| `/dbc:implement <ID>` | Sonnet | Implementa un plan leyendo solo el plan |
| `/dbc:next [carril]` | el de la sesión | Siguiente tarea de la cola cuando no tiene plan |
| `/dbc:design-by-contract …` | el de la sesión | Proyecto nuevo, adoptar, cambiar el contrato (`contrato …`), auditoría (`auditoría`, `completitud`), integraciones |

Flujo de trabajo:

```bash
claude --model opus     # → /dbc:plan B01 B02 B03
claude --model sonnet   # → /dbc:implement B01   (/clear entre tareas)
```

## Cambiar el plugin

Edita `plugins/dbc/skills/…`, sube `version` en `plugins/dbc/.claude-plugin/plugin.json`,
valida con `claude plugin validate plugins/dbc` y haz commit + push. Los demás actualizan con
los comandos de arriba.
