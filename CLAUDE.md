# CLAUDE.md — cotizador-ayj-v2 (MQ V2, desarrollo activo)

## AYJ Maquinaria — Ecosistema de Cotizadores

Este repo es parte de un ecosistema de 5 cotizadores para AYJ Maquinaria S.A.S. (35+ años, Bogotá y Medellín). Cada uno es un **single-file HTML** (~5.000-6.000 líneas, HTML+CSS+JS vanilla, sin frameworks) que se despliega directo con GitHub Pages.

| Cotizador | Repo | Estado |
|---|---|---|
| MQ V1 — Máquinas (estable) | aristi01AYJ/cotizador-ayj | producción |
| MQ V2 — Máquinas (desarrollo) | aristi01AYJ/cotizador-ayj-v2 | desarrollo activo |
| SW — Software imos | aristi01AYJ/cotizador-software | producción |
| ADH — Adhesivos | aristi01AYJ/cotizador-adhesivos | producción |
| SVC — Servicios | aristi01AYJ/cotizador-service | producción |

**Regla de sincronización:** en Máquinas, V2 es donde se desarrolla y prueba; cuando un fix o feature está estable, se replica en V1 (mismo cambio, mismo archivo `index.html`).

### Stack técnico
- Frontend: HTML + CSS + JS vanilla, un solo archivo `index.html` por cotizador
- Auth: MSAL.js + Azure AD (Entra ID)
- Backend de datos: Microsoft Graph API v1.0 → listas de SharePoint Online
- Hosting: GitHub Pages
- TRM: BanRep (datos.gov.co), con fallback a open.er-api.com (ADH usa BanRep × 1.15)

### ⚠️ Credenciales — NUNCA hardcodear en este repo (es público)
Este repositorio es **público** (GitHub Pages gratis). El token de GitHub, el Azure AD App ID, los `siteId` de SharePoint y la clave del módulo MG viven en el documento de handoff privado del Proyecto de Claude **"APP PARA AYJ - COTIZADOR"** — pídeselos a Adolfo o consulta ese documento, nunca los pegues en código ni los subas a este repo. Si necesitas guardarlos localmente para trabajar, usa un `.env` o similar que esté en `.gitignore`.

### Workflow de edición de código
1. Editar `index.html` localmente (o vía Claude Code con el repo clonado).
2. **Validar el JS antes de cualquier commit**: extraer el bloque `<script>...</script>` principal y correr `node --check`. Nunca subir sin este paso.
3. Commit descriptivo, push a `main` (GitHub Pages se actualiza solo).
4. Si el fix aplica también a la otra versión de Máquinas (V1 ↔ V2), replicarlo ahí igual.
5. Actualizar el handoff del Proyecto de Claude con el fix aplicado (fecha, archivo, línea, qué cambió) para que la próxima sesión tenga contexto.

### Reglas críticas Graph API / SharePoint (aplican a todos)
- NUNCA usar `$select=fields` en queries → error 400. Usar `expand=fields&$top=500`.
- Campo NIT interno: `NIT_x002f_RUT` — leer con `.toString().trim()`.
- Campos tipo Hyperlink (LinkPDF, Ficha Técnica): `{Url:"...", Description:"..."}`.
- Clientes/Contactos: SIEMPRE en el `siteId` de Comercial (todos los cotizadores, incluso SVC).
- **SIEMPRE usar `graphGetAll()` (pagina con `@odata.nextLink`) para cualquier query que pueda superar 500 ítems** — especialmente `generarNumOferta()`. `graphGet()` con `$top=500` solo trae la primera página; una vez la lista de Cotizaciones supera 500 registros, el consecutivo se queda pegado para siempre porque deja de ver los ítems más nuevos (bug real, 31-ago-2026, afectó los 5 cotizadores).


## Este repo específico: Máquinas V2

Es la versión de **desarrollo activo**. Aquí se prueban los cambios primero; cuando quedan estables, replicarlos en `cotizador-ayj` (V1) — mismo archivo, mismo fix.

### Módulo Liquidador (cotización de máquinas)
- Dos modos: `COSTO_AYJ` (puesto en AYJ) y `FOB` (desde fábrica).
- `precioTarifa = PVN = totalCostosUSD / (1 - margen)` — **no** sumar instalación otra vez, ya está incluida en `totalCostosUSD`.
- El descuento al cliente se define SOLO en el carrito, nunca en el liquidador.
- `liqSeleccionarMaquina` no debe sobreescribir CIF si ya hay un valor > 0.
- `liqSyncInstMaquina` no debe recalcular instalación si `liqInstalacion` > 0.
- Snapshots al agregar un ítem liquidado: `item._liqSnapshot`, `item._instSnapshot`, `item._finSnapshot` — se usan para restaurar el liquidador al editar (`liqEditarItem`).
- Desglose de Costos → fila "Otros" muestra `Otros — <descripción>` leyendo `liqOtrosDesc` (fix ago-2026).

### Módulo de Márgenes (MG)
- Acceso con clave (ver handoff, no está en este repo).
- **Importante:** `item.tarifa` ya es el total para la cantidad (ver `cambiarCantCarrito`) — en `mostrarResumenMargenes()` NO volver a multiplicar `tarifa` por `cant`. `costoAYJ` sí se guarda por unidad y sí se multiplica por `cant`. (Bug corregido ago-2026 — si ves un margen que no cuadra al usar cantidad > 1, revisa esto primero.)
- `descBase` (el % de descuento sin el +3% promocional) debe definirse explícitamente en `guardarCotizacion()`, no solo calcularse en `calcTotales`.
- Texto de descuento es dinámico vía `getCausalEspecial()` — no usar texto fijo.

### Feature: ítem incluido
- `item.incluidoEn` = id del ítem padre. El padre muestra `tarifa + sumHijos`. `calcTotales` debe ignorar los ítems incluidos en el subtotal directo.

### Fixes recientes
- **31-ago-2026 — Consecutivo de cotización pegado.** `generarNumOferta()` usaba `graphGet()` sin paginar; se cambió a `graphGetAll()`. Ver regla en la sección de Graph API arriba. Replicar en V1 si no está ya.

### Extracción del JS para `node --check`
```js
s = content.find('<script>\nconst CLIENT_ID')
e = content.rfind('\n</script>')
js = content[s+8:e]
```
