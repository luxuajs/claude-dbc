---
name: implement
description: Implementa un plan de tarea (plans/<ID>.md) de un proyecto design by contract leyendo solo el plan.
argument-hint: <ID>
model: sonnet
disable-model-invocation: true
---

# Implementar plans/$ARGUMENTS.md

1. Lee `plans/$ARGUMENTS.md` y **solo** los archivos que nombra. Ni `PROGRESO.md`, ni el
   contrato, ni otros planes, ni `contracts/dist/` o `contracts/generated/`.
2. Sigue los **Pasos** en orden. No rediseñes, no amplíes el alcance, no añadas
   dependencias que el plan no nombre.
3. Código: KISS, sin abstracciones que el plan no pida; comentarios en español, breves, que
   digan qué hace y por qué; cumple la sección **Seguridad** del plan.
4. Si algo choca con el contrato o el plan no se puede cumplir: **para**. Escríbelo en
   *Bloqueos* del plan (qué, dónde, por qué), haz commit de lo hecho y termina.
5. **Verificación** y **Cierre** exactamente como dice el plan. Si la verificación falla y no
   lo puedes arreglar dentro del plan, es un bloqueo.
6. Responde en 3 líneas: qué se hizo, cómo se verificó, si hubo bloqueo.
