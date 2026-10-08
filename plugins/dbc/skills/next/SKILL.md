---
name: next
description: Avanza la siguiente tarea de la cola (PROGRESO.md) de un proyecto design by contract cuando no tiene plan.
argument-hint: "[carril o ID]"
disable-model-invocation: true
---

# Siguiente tarea $ARGUMENTS

1. `PROGRESO.md` raíz → la tarea *En curso* del carril pedido, o la primera pendiente.
   **Solo esa.** Si tiene `plans/<ID>.md`, haz lo que dice `/dbc:implement` con ese plan y para.
2. El `PROGRESO.md` de su carpeta y lo que la tarea indica en "Leer". Para una operación:
   `grep -n "operationId: <op>" contracts/<servicio>/paths.yml`, solo ese bloque y sus
   esquemas. Nunca `contracts/dist/` ni `contracts/generated/`.
3. Si no cabe en una sesión, pártela en la cola y haz la primera parte. Márcala *En curso*.
4. Implementa con el estilo de `${CLAUDE_PLUGIN_ROOT}/skills/design-by-contract/references/estilo.md`
   (léelo si vas a escribir código). Si choca con el contrato: para y cambia primero el
   contrato (`/dbc:design-by-contract contrato …`); la tarea queda *bloqueada por* esa.
5. Verifica con su "Hecho cuando" (y `pnpm typecheck && pnpm humo` si tocaste código). Si
   falla, no está hecha.
6. Cierra: a *Hecho* con fecha, `PROGRESO.md` de la carpeta (los `operationId` en servicios),
   siguiente indicada, commit. Responde en 3–5 líneas.
