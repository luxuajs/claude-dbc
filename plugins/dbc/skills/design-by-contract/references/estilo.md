# Estilo de código y seguridad

Lo que el que planifica aplica al escribir los **Pasos** de un plan, y lo que el que
implementa respeta. El que planifica no copia este archivo al plan: copia solo las reglas y
los riesgos que tocan a esa tarea.

## Principios (en caso de duda, gana KISS)

- **KISS.** La solución más simple que cumple el contrato. Una función antes que una clase,
  una clase antes que una jerarquía.
- **DRY, con una sola fuente de verdad.** El contrato valida la entrada: no se repite esa
  validación a mano. Los errores salen del catálogo (`common/errores.yml`). Se abstrae a la
  **tercera** repetición, no a la segunda.
- **SOLID donde hay variación real, no "por si acaso":**
  - **S** — manejador delgado (traduce HTTP ↔ dominio) → servicio de dominio (reglas) →
    repositorio (datos). Un archivo, una razón para cambiar.
  - **O / D** — lo que cambia de proveedor (correo, pagos, voz, archivos) se usa a través de
    una función o puerto nuestro y se inyecta; cambiar de proveedor es añadir un adaptador,
    no tocar el dominio.
  - **L / I** — interfaces pequeñas, solo con lo que usa quien las consume.
  - Una interfaz con una sola implementación solo si hace falta para probar o para el
    límite con un proveedor; si no, sobra.

## Patrones que encajan en este stack

| Patrón | Dónde | Cuándo no |
|---|---|---|
| **Adapter** | Proveedores externos y webhooks (`x-adaptador`) | Código propio sin frontera externa |
| **Strategy** | Elegir proveedor o regla por configuración (`CORREO_PROVEEDOR`) | Cuando hay una sola opción |
| **Repository** | Acceso a datos por agregado, encima de Prisma | Una consulta suelta que no se reutiliza |
| **Factory** | Montar el servicio con sus dependencias (`crearServicio`) | Crear objetos simples |
| **Result / errores tipados** | Errores esperados del dominio → `type` del catálogo | Fallos no controlados (esos van al manejador global) |

Ningún patrón se añade para "dejarlo preparado". Si el plan pide uno, dice por qué.

## Comentarios

- En **español**, breves: qué hace y, si no es obvio, **por qué**. Una línea casi siempre basta.
- Cada función exportada lleva un JSDoc de una línea. Lo obvio no se comenta.
- Un atajo deliberado se marca con su límite y su salida: `// atajo: … ; cuando … , …`.
- Los identificadores siguen la convención que ya tiene el repo.

## Seguridad — OWASP Top 10:2025

El plan lista **solo los riesgos que tocan a su tarea** y cómo se cubren. Lo que ya cubre la
plataforma (cargador, middlewares, contrato) se nombra, no se reimplementa.

| Riesgo | Qué exige aquí |
|---|---|
| **A01 Broken Access Control** (incluye SSRF) | Tenant y rol desde el token, nunca de la petición; recurso ajeno → 404; `x-roles` → 403. Ninguna URL saliente construida con datos del usuario sin lista blanca. |
| **A02 Security Misconfiguration** | CORS solo para dominios propios; cabeceras de seguridad; sin trazas ni detalles internos en errores; nada de claves por defecto. |
| **A03 Software Supply Chain Failures** | Dependencias justificadas y fijadas en el lockfile; `pnpm audit` en CI; nada de paquetes para lo que caben pocas líneas. |
| **A04 Cryptographic Failures** | Secretos solo en variables de entorno; tokens opacos guardados con hash; comparación en tiempo constante para firmas; TLS siempre. |
| **A05 Injection** | Solo consultas parametrizadas (Prisma, nunca SQL concatenado); el contrato valida tipos y patrones; datos del usuario que entran a un prompt de IA, limpiados y acotados. |
| **A06 Insecure Design** | Límites de negocio en el contrato (cupos, tamaños, duraciones); idempotencia en lo que cuesta dinero; topes de gasto en proveedores. |
| **A07 Authentication Failures** | Sesión verificada por firma y caducidad (JWKS); webhooks con la firma de su proveedor sobre el cuerpo en crudo; rutas públicas con captcha y límite por IP. |
| **A08 Software or Data Integrity Failures** | Nada sin firma entra al sistema (webhooks, colas); deserializar solo lo que el contrato declara. |
| **A09 Security Logging & Alerting Failures** | Log JSON con `request_id`, `tenant_id` y resultado en accesos denegados y fallos de firma; nunca secretos, tarjetas ni datos personales en logs. |
| **A10 Mishandling of Exceptional Conditions** | Fallar **cerrado**: ante un error, denegar, no permitir. Errores esperados → `type` del catálogo; inesperados → 500 genérico y log. Timeouts en toda llamada externa. |

Fuente: <https://owasp.org/Top10/2025/>
