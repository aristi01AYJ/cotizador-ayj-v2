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

### Feature: StockIlimitado (evitar falso "ÚLTIMA UNIDAD")
- El badge/label "ÚLTIMA UNIDAD" se dispara cuando `grupo.stockCount===1` (cuenta filas en SharePoint con `ESTADO_`=Disponible para ese título) — para ítems físicos serializados tiene sentido, pero para software/licencias (ej. `DESPIECE_PRO`) que solo tienen 1 fila por catalogación, es un falso positivo.
- Columna SharePoint nueva (Sí/No): **`StockIlimitado`**. Se lee en `cargarMaquinas()`, se propaga en `procesarCatalogo()` (OR entre todas las filas del mismo título), y se arrastra al carrito en `agregarGrupo()`, `liqGetItem()` y al reconstruir el carrito al editar una oferta guardada.
- Las 2 condiciones que deciden mostrar el badge (`renderCarrito()` y `generarPDF()`, más la tarjeta de catálogo) ahora exigen `!item.stockIlimitado` además de `stockCount===1`.
- Si la columna no existe todavía en SharePoint, `f.StockIlimitado` llega `undefined` → se comporta exactamente igual que antes (sin romper nada).

### Feature: anti-duplicados al guardar + Acciones del Historial
- `guardarCotizacion()` siempre hacía POST (nunca update), y tras guardar dejaba el mismo N° de oferta en pantalla a propósito ("el usuario debe pulsar Nueva Oferta"). Guardar dos veces sin pasar por "Nueva Oferta" (típico: ajustar algo en el Liquidador y volver a Guardar) creaba un **registro duplicado idéntico**.
- Fix: si `!modoEdicion`, antes de guardar se consulta SharePoint por si el número actual ya existe; si existe, se convierte automáticamente en revisión (`-R2`, `-R3`...) con el mismo esquema que ya usaba `editarOferta()`. Nunca deben quedar dos registros con el mismo `Title`/`ID_SYNERGY`.
- **Acciones del Historial** (orden fijo): PDF · Duplicar · Editar · ✓ Ganada · ✗ Perdida · Eliminar · ★★★★★ · "···" (Enviada/En negociación vía `cambiarEstado()`, se mantiene para los 2 estados que no tienen botón dedicado). Ganada/Perdida usan `aplicarEstado()` directo (un clic, sin modal).
- **Probabilidad** ahora son 5 estrellas clicables (`estrellasHistorial()` + `editarProbabilidadEstrellas()`), reemplazando el `prompt()` de 0-100%. Se sigue guardando 0-100 en pasos de 20 en el campo `Probabilidad` de SharePoint (no se cambió el esquema de almacenamiento, solo la UI) — el Dashboard sigue leyendo el mismo campo sin cambios.

### Feature: limpieza de duplicados existentes
- Botón "Duplicados" en el Historial (clave `CLAVE_MG`, `abrirLimpiezaDuplicados()`): dos pasadas de detección, lógica compartida en `_buscarDuplicados()`.
  1. **Exactos**: agrupa por N° de oferta normalizado (`_normNumOferta` — trim + mayúsculas + colapsa espacios, para no fallar por diferencias invisibles de texto). Una revisión `-R2` es un Title distinto, NO se agrupa como duplicado — es intencional. Se borran en bloque con confirmación única.
  2. **Posibles** (solo sobre lo que no cayó en un grupo exacto): mismo Cliente + mismo Total pero N° distinto — candidato a duplicado con número corrido a mano o donde `ID_SYNERGY` no coincide con `Title`. Se listan en amarillo, **nunca se borran en bloque** — botón individual por ítem, señal más débil que amerita revisión caso por caso.
  - En ambas pasadas se conserva el de mayor `id` de SharePoint (= más reciente). Complementa el fix de anti-duplicados: ese evita duplicados nuevos, esto limpia los que ya existían de antes.

### Feature: Vista Previa en el Historial
- Bajo el nombre del cliente, una línea pequeña con los nombres de los ítems separados por " | " (`previewItems()`, ej. "G3 | EP5 (Liq) | CRG3"), leídos de la columna `Items` ya guardada en cada cotización. Trunca a ~60 caracteres con `title` (tooltip) mostrando el listado completo. Pensado para distinguir varias ofertas al mismo cliente sin abrir cada PDF.
- **Origen LIQ/TAR:** cada ítem guardado (`guardarCotizacion` → `fields.Items`) incluye `origen:'LIQ'` si `item.id` empieza con `liq_` (viene del Liquidador) o `'TAR'` si es precio de catálogo normal. `previewItems()` marca los LIQ con "(Liq)" junto al nombre. Ofertas guardadas antes de este cambio no traen `origen` — no se les inventa, simplemente no muestran la marca.

### Feature: Estado como dropdown rápido (3 estados)
- La columna Estado del Historial es un `<select>` inline (coloreado igual que la píldora anterior) con 3 opciones: Enviada / ✓ Ganada / ✗ Perdida. Cambiar de estado es un clic + seleccionar, sin modal.
- `'En negociación'` se retiró como opción nueva pero **sigue existiendo en `ESTADOS_CONFIG`** (no se borró) — un registro histórico con ese estado se sigue coloreando bien y aparece como opción extra ya seleccionada en su propio `<select>` (no se fuerza a cambiarlo), y sigue contando en el gráfico "Pipeline por Estado" del Dashboard, que itera `Object.keys(ESTADOS_CONFIG)`. Si se necesita quitarlo del todo, hay que revisar primero ese gráfico.
- Como el estado ahora se cambia desde la propia columna Estado, se retiraron de **Acciones** los botones que quedaban redundantes: ✓ Ganada, ✗ Perdida, y el "···" (Cambiar estado / modal). Acciones quedó: PDF · Duplicar · Editar · Eliminar · Estrellas. La función `cambiarEstado()` (el modal viejo) queda sin usar en la UI pero no se borró.

### Feature: gráficos interactivos (cross-filter) en Inteligencia Comercial
- Los 4 gráficos del dashboard (`renderChart()`, Chart.js) aceptan un `tipoFiltro` opcional (`'vendedor'|'estado'|'tipologia'`) que activa `onClick`: clic en un segmento/barra llama `aplicarFiltroDash(tipo, valor)`, que filtra `dashData` y vuelve a llamar `renderDashboard()` — como los 4 gráficos y los KPIs se derivan todos de la misma variable `datos` filtrada, el cross-filter llega a todo junto sin lógica extra por gráfico.
- Clic de nuevo sobre el mismo segmento quita el filtro (toggle). Banner `#dashFiltroActivo` visible mientras hay filtro, con botón "Quitar filtro" (`limpiarFiltroDash()`).
- `dashFiltro` se resetea automáticamente al recargar (`cargarDashboard()`, botón refresh o cambio de año) — nunca se arrastra un filtro viejo entre cargas.
- El filtro de tipología es a nivel de OFERTA (si la oferta tiene al menos un ítem de esa tipología, entra en el filtro) — el total de la oferta que se cuenta sigue siendo el total completo, no solo la parte de esa tipología.

### Fixes recientes
- **12-sep-2026 — Gráficos interactivos (cross-filter) en Inteligencia Comercial.** Ver sección arriba.
- **12-sep-2026 — Origen LIQ/TAR en Vista Previa + Estado como dropdown de 3 opciones.** Ver secciones arriba.
- **12-sep-2026 — Vista Previa (nombres de ítems) en el Historial.** Ver sección arriba.
- **12-sep-2026 — Detección de duplicados más tolerante + segunda pasada Cliente+Total.** La comparación exacta de Título dejaba pasar duplicados con diferencias de mayúsc/espacios, o donde el N° de oferta no coincidía pero claramente era el mismo negocio (mismo cliente, mismo total). Ver sección arriba.
- **12-sep-2026 — Herramienta para limpiar duplicados ya existentes.** Ver sección arriba.
- **12-sep-2026 — Cotizaciones duplicadas al re-guardar + Acciones mejoradas.** Ver sección arriba. Aplicado también en V1.
- **10-sep-2026 — Falso "ÚLTIMA UNIDAD" en ítems de stock ilimitado.** Ver sección arriba. Aplicado también en V1.
- **31-ago-2026 — Consecutivo de cotización pegado.** `generarNumOferta()` usaba `graphGet()` sin paginar; se cambió a `graphGetAll()`. Ver regla en la sección de Graph API arriba. Replicar en V1 si no está ya.

### Extracción del JS para `node --check`
```js
s = content.find('<script>\nconst CLIENT_ID')
e = content.rfind('\n</script>')
js = content[s+8:e]
```
