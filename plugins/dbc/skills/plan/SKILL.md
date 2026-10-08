---
name: plan
description: Escribe planes de tarea (plans/<ID>.md) en un proyecto design by contract para que otro modelo los implemente leyendo un solo archivo.
argument-hint: <ID> [<ID>…]
model: opus
disable-model-invocation: true
---

# Planificar: $ARGUMENTS

Escribes planes que **otro modelo implementa leyendo solo el plan**. El ahorro existe si
el plan trae dentro todo lo necesario; si dice "lee X", se pierde.

## Antes de planificar

1. **Revisa lo último implementado:** `git log --oneline -5` y `git show --stat` del último
   commit de implementación; si un plan tiene algo en *Bloqueos*, resuélvelo primero (cambio
   de contrato con `/dbc:design-by-contract contrato …`) y actualiza ese plan.
2. Si no existe `plans/_template.md`, créalo desde `${CLAUDE_PLUGIN_ROOT}/skills/design-by-contract/references/plantillas.md`
   § 17 (`grep -n "^## 17"` y solo ese tramo) y añade a `CLAUDE.md` el párrafo de planes (§1).

## Por cada ID de la tanda

Lee una vez lo común y después, por tarea, solo lo suyo:

1. Su entrada en `PROGRESO.md` (`grep -n "\*\*<ID> ·" PROGRESO.md`) y las reglas de su carril.
2. Sus operaciones: `grep -n "operationId: <op>" contracts/<servicio>/paths.yml` y solo ese
   bloque y sus esquemas. **Nunca** `contracts/dist/` ni `contracts/generated/`.
3. Los archivos de código que va a tocar, solo las partes relevantes.
4. Estilo y seguridad: `${CLAUDE_PLUGIN_ROOT}/skills/design-by-contract/references/estilo.md`. Copia al
   plan solo las reglas y los riesgos OWASP que tocan a esta tarea.

Escribe `plans/<ID>.md` desde la plantilla con:
- **Archivos** con rutas exactas, y **Contrato aplicable** con los YAML **copiados**.
- **Pasos** concretos; en lo delicado (seguridad, SQL, firmas, reglas de negocio), el código
  exacto con sus comentarios en español. Patrones solo donde `estilo.md` lo justifica.
- **Seguridad**: riesgos OWASP de la tarea y cómo quedan cubiertos.
- **Verificación** con comandos y salida esperada, y **Cierre** con el mensaje de commit.

Si una tarea no cabe en una sesión, pártela en la cola (`B04a`, `B04b`) y planifica cada parte.

## Al terminar

Commit de los planes (`plans: <IDs>`). Responde en 3–5 líneas: qué planes hay, cuál va
primero y cómo arrancar: `claude --model sonnet` → `/dbc:implement <ID>`.
