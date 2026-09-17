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
- El filtro de tipología es a nivel de OFERTA (si la oferta tiene al menos un ítem de esa tipología, entra en el filtro) — el total de la oferta que se cuenta sigue siendo el total completo, no solo la parte de esa tipología. También soporta filtrar por `'cliente'` (desde el gráfico Top 15 Clientes).
- **10 gráficos en total:** Pipeline por Estado, Ventas por Asesor (solo Ganadas), Ofertas x Asesor Cantidad (todos los estados), Ofertas x Asesor USD (todos los estados), Tasa de Cierre por Asesor, Top Tipologías, Probabilidad Promedio x Vendedor (pipeline abierto), Probabilidad x Valor (scatter, pipeline abierto), Top 15 Clientes por N° de ofertas, Origen (Stock/Importación/Bajo Pedido). El scatter (`renderScatterProbValor()`) es aparte de `renderChart()` porque usa ejes numéricos x/y en vez de labels por categoría.
- **Origen** usa `item.tipo` (`'stock'|'importacion'|'pedido'`, disponibilidad al agregar al carrito) vía el helper `origenLabel()` — a diferencia del bug de Tipologías (12-sep-2026), aquí `item.tipo` es exactamente lo que se quiere mostrar, no un fallback equivocado.
- `asesoresMap` (calculado una vez, reutilizado por 3 gráficos) trackea `{total, ganadas, cantidad}` por asesor — `total`/`cantidad` son TODOS los estados, `ganadas` solo Cerrada-Ganada. Si se agrega un cuarto gráfico "por asesor", reutilizar este map en vez de recalcular.

### Filtros de barra superior: Año, Mes, Asesor, "Ocultar datos incompletos"
- `dashMes` (dropdown Enero-Diciembre) y `dashAsesor` (poblado dinámicamente por `poblarAsesoresDash()`, se refresca en cada `cargarDashboard()`) son filtros de verdad en la barra, no solo accesibles por clic. Se aplican en `renderDashboard()` ANTES del cross-filter por clic.
- Clic en cualquier barra "por asesor" (Ventas, Cantidad, USD, Tasa de Cierre, Probabilidad) mueve el dropdown `dashAsesor` en vez de pasar por `dashFiltro`/el banner — `aplicarFiltroDash()` trata `'vendedor'` como caso especial. Así hay un solo control visible para "qué asesor", sea que se elija del dropdown o con clic.
- **`dashOcultarIncompletos`** (checkbox, activado por defecto): a nivel de oferta excluye las que no tienen `cliente` o están en `total<=0` (dato roto, no negocio real); a nivel de ítem excluye "Sin tipología" SOLO del gráfico Top Tipologías (la oferta sigue contando en el resto). Es puramente un filtro de vista — no borra ni modifica nada en SharePoint; destildarlo muestra todo, útil para auditar qué se está ocultando. Se agregó porque reparar retroactivamente el histórico de "Sin tipología" no es viable (ver fix de 12-sep-2026 sobre el nombre interno del campo) — este toggle evita que ese hueco de datos viejo distorsione el dashboard sin intentar "arreglar" lo histórico.

### ⚠️ Cuidado al tocar `item.tipo` vs `item.tipologia` en cualquier agregación
`item.tipo` es **disponibilidad** (`'stock'|'importacion'|'pedido'`), NO una categoría de máquina. Usarlo como fallback de tipología (`item.tipologia||item.tipo`) mete valores como "pedido" en un chart de tipologías — bug real, corregido 12-sep-2026. El fallback correcto es un string fijo tipo `'Sin tipología'`.
También: `item.tarifaUSD` (o `item.tarifa` en el carrito) **ya es el total de la línea** (tarifa × cantidad se aplicó al agregar al carrito) — jamás volver a multiplicar por `cantidad` en una agregación. Mismo patrón de bug que el de Márgenes (20-ago-2026) y ahora el de Top Tipologías (12-sep-2026); si aparece un tercero, es la misma familia de error.

### Por qué salían TANTAS "Sin tipología" — bug real, ya corregido
Cuando la lista mostraba de "Sin tipología" nombres de catálogo reales (GAMA 1-4, EP5, EP10, CR_G2...) que claramente sí tienen Tipología asignada en SharePoint, la causa no era falta de dato — era que `cargarMaquinas()` nunca estaba leyendo el campo. `f.Tipologia` (sin tilde) casi nunca es el nombre interno real: cuando la columna se crea como "Tipología", SharePoint suele codificar la í como `_x00ed_` — el mismo patrón que ya maneja este archivo para Ubicación (`Ubicaci_x00f3_n`) y Descripción (`Descripci_x00f3_n`). La detección dinámica vía `/columns` debía rescatar esto pero fallaba en silencio (`catch(e){}` sin avisar). **Fix (12-sep-2026):** fallback explícito `f.Tipolog_x00ed_a`, el `catch` ya no traga el error (queda en consola), y se agregó un log de diagnóstico (`console.log('[AYJ] tipologiaField detectado...`) que imprime qué campo se detectó y qué claves con "tip" trae el primer ítem — útil si esto vuelve a fallar con otro nombre interno.
Causas legítimas de "Sin tipología" que SÍ pueden seguir apareciendo tras este fix: ítems agregados manualmente vía Liquidador ("no está en el catálogo" — `liqGetItem()` deja `tipologia:''` a propósito) y cotizaciones históricas guardadas antes de este fix. Al hacer clic en "Sin tipología" en el gráfico Top Tipologías, el banner de filtro muestra debajo (`#dashFiltroDetalle`) el nombre exacto de cada ítem afectado, deduplicado.

### ⚠️ Jorge Pulido vs Jorge Salamanca — usar SIEMPRE `asesorLabel()`, nunca `nombre.split(' ')[0]`
Hay dos vendedores con el mismo primer nombre ("Jorge Pulido" y "Jorge Salamanca"). Cualquier código que agrupe/compare vendedores por `nombre.split(' ')[0]` los mezcla como si fueran la misma persona — esto afectó gráficos de Inteligencia, el filtro de Vendedor del Historial, y (más grave) los **chequeos de permisos** de "es tu propia oferta" (`currentUserName.includes(vendedor.split(' ')[0])`), que dejaban a Jorge Pulido editar/borrar ofertas de Jorge Salamanca y viceversa. Fix (12-sep-2026): función `asesorLabel(nombre)` (mismo criterio que `vendedorFolderName()`, ya existente para nombrar archivos) devuelve `'Jorge P'`/`'Jorge S'` para esos dos, primer nombre normal para el resto — usada en TODOS los puntos de agrupación/comparación por vendedor (dashboard, Historial, permisos). Si se agrega un nuevo lugar que agrupe por vendedor, usar `asesorLabel()`, no `.split(' ')[0]` ni `.includes()` por substring de nombre.

### Feature: Gerente de Cuenta en Clientes ("Mis Clientes" / "Todos")
- Cada cliente tiene un dueño (`c.asesor`, campo SharePoint `Asesor`, mismo campo que ya existía en Cotizaciones). Se registra automáticamente al **crear** el cliente (`guardarCliente()`: `if(!clienteEditandoId) fields.Asesor=currentUserName`) — editar un cliente existente ya NO pisa el Asesor, a diferencia de antes.
- Checkbox "Mis Clientes" en la barra de Clientes (`#clientesVerTodos`, default sin marcar = ver Todos): filtra `renderTablaClientes()` por `asesorLabel(c.asesor)===asesorLabel(currentUserName)` — usa `asesorLabel()`, no comparación directa de string, por el mismo motivo que el fix de Jorge P/Jorge S de abajo.
- Clientes creados **antes** de este cambio no tienen Asesor. El panel de detalle (`seleccionarClienteDetalle()`) muestra "⚠ Sin Gerente de Cuenta" + botón **Asignarme** (`asignarmeCliente(idx)`) cuando `c.asesor` está vacío — acción explícita de un clic que hace PATCH a SharePoint y solo actúa si el cliente de verdad no tenía dueño (si ya tiene uno, el botón ni aparece).
- **Asignar/reasignar a otra persona** (no solo a uno mismo): botón "Asignar a otro" (sin dueño) o "Reasignar" (ya tiene dueño) → `mostrarReasignar(idx)` despliega un `<select>` inline con `listaAsesoresConocidos()` (nombres completos ya vistos como `Asesor` de algún cliente o `Vendedor` de alguna oferta, más el usuario actual — deduplicados por `asesorLabel()`, así Jorge Pulido/Jorge Salamanca quedan como 2 opciones, no una) excluyendo al dueño actual. `confirmarReasignar(idx)` confirma con mensaje distinto según si es primera asignación o transferencia, y llama al PATCH compartido `patchAsesorCliente(idx, nombre)` — el mismo que usa `asignarmeCliente()`, para no duplicar la llamada a Graph. **No tiene gate de permisos** (igual que `eliminarClienteIdx` — es una herramienta interna de equipo pequeño, no un sistema con roles).

### Feature: gráfico "Ofertas x Mes" en Inteligencia Comercial
- Nuevo card full-width arriba de "Pipeline por Estado" (`#chartOfertasMes`, barra, cuenta ofertas por mes calendario del año filtrado — no suma dinero, solo cantidad). Usa el array `MESES_CORTO` (Ene..Dic) como eje y helper.
- Clic en una barra activa el filtro de mes: `aplicarFiltroDash('mes', valor)` es un caso especial más (como `'vendedor'`) que mueve el dropdown `dashMes` de la barra superior en vez de pasar por el banner de `dashFiltro` — toggle: clic de nuevo en el mismo mes lo quita.

### Feature: overlay de carga bloqueante al iniciar sesión
- `#appLoadingOverlay` (fullscreen, z-index 9999) se muestra en `showApp()` apenas se ve `#appScreen` y se oculta solo cuando `resolverSite()` + `Promise.all([cargarMaquinas(),cargarClientes(true),cargarTRM()])` + `generarNumOferta()` terminan (bloque `try/finally` — se oculta también si algo falla, para no dejar al usuario atrapado con el spinner). Antes se podía interactuar con la app (agregar al carrito, etc.) mientras el catálogo aún estaba cargando o vacío.

### Feature: Ofertas del cliente + estadísticas en su ficha
- Dentro de `seleccionarClienteDetalle()`, entre el header y Contactos: nueva tarjeta "📊 Ofertas a este cliente" que cruza `historialData` filtrando por `r.cliente` (match case-insensitive/trim, mismo criterio que `cargarContactosPorCliente`) contra `c.titulo`.
- Si `historialData` aún no se cargó en la sesión (el usuario entró directo a Clientes sin pasar por Historial), se carga ahí mismo bajo demanda (`if(!historialData.length) await cargarHistorial()`) — una sola carga, se reutiliza después.
- KPIs (Ofertas, Total Cotizado, Ganadas, Tasa de Cierre) + gráfico de dona por Estado (`#chartClienteEstado`, mismos colores de `ESTADOS_CONFIG`, sin cross-filter — es un chart aislado de la ficha, no toca `dashData`/`renderDashboard()`) + listado de las ofertas (fecha, N°, estado, total).
- Botón **"Ver en Historial"** (`verOfertasClienteEnHistorial(nombreCliente)`): salta a la vista Historial con `#hFiltCliente` ya lleno con el nombre del cliente, para ver/editar cada oferta sin retipear el filtro.
- Cliente sin ofertas: mensaje "Sin ofertas registradas para este cliente", sin KPIs/gráfico/botón (nada que mostrar).

### Feature: agrupar revisiones (-R2, -R3...) en las estadísticas — "Solo vigentes"
- Reabrir y volver a guardar una oferta (botón Editar, o el resguardo anti-duplicado de `guardarCotizacion()`) SIEMPRE crea un registro nuevo en SharePoint con sufijo `-Rn` — nunca sobreescribe el anterior, a propósito (para no perder el historial de qué se cotizó antes). El problema: una misma negociación real puede terminar con 3-4 filas, y para Inteligencia/estadísticas debía contar como 1 sola — antes se contaban todas por separado, inflando "N° Ofertas" y "Total Cotizado".
- Solución: 3 funciones puramente de vista (no tocan SharePoint) — `ofertaBaseNum(num)` (quita el sufijo `-Rn`), `ofertaNumRevision(num)`, y `soloVigentes(lista)` (agrupa por N° base y se queda con la revisión de mayor N°, desempatando por mayor id de SharePoint = más reciente). Funciona igual sobre `historialData` y `dashData` (ambas traen `.num` y `.id`).
- Aplicado en 3 lugares para que cuenten igual: `renderDashboard()` (Inteligencia — se agrupa ANTES de filtrar por año, así la vigente cuenta en SU año, no en el de una revisión vieja), `filtrarHistorial()` (Historial — nuevo checkbox **"Solo vigentes"**, marcado por defecto, junto a "Mis ofertas"), y las estadísticas de "Ofertas a este cliente" en la ficha del cliente.
- El Historial NO oculta el historial completo: con "Solo vigentes" marcado, cada fila con más de 1 revisión muestra un badge clicable "Nrev" (`verRevisionesFamilia(baseNum)`) que destilda el checkbox y filtra por ese N° base, mostrando las revisiones completas con un clic.
- Decisión de diseño (contra las 2 ideas alternativas que planteó el usuario): NO se cambió el comportamiento de guardar (seguir creando -Rn siempre, nunca sobreescribir) porque eso fue justo lo que se corrigió antes para no perder historial; y NO se escribe ningún Estado tipo "Cerrada-Actualizada" a SharePoint en las revisiones viejas, porque el campo Estado ya alimenta mucha lógica existente (Pipeline, Ganadas/Perdidas por asesor, dropdown de cambio de estado) y forzaría a excluir ese estado en todos esos sitios — el filtro de vista logra lo mismo sin tocar datos ni arriesgar esa lógica.

### ⚠️ Requiere columna nueva en SharePoint: `FechaCierre` (lista Cotizaciones)
Feature "Posible cierre + Seguimiento" (abajo) necesita una columna **`FechaCierre`** (tipo Fecha, solo fecha sin hora) en la lista **Cotizaciones** de SharePoint. Nombre exacto `FechaCierre` (sin tildes/espacios) a propósito — así el nombre interno de SharePoint coincide con el nombre visible y no hay riesgo del problema de codificación `_x00XX_` que ya afectó a Tipología/Ubicación.
**Leer** sin esa columna no falla (`f.FechaCierre` llega `undefined`, el selector queda en "— Sin definir —"). **Guardar SÍ fallaba** — bug real reportado por el usuario 17-sep-2026: Graph rechaza el POST completo con `400 invalidRequest: "Field 'FechaCierre' is not recognized"` en cuanto se elegía cualquier "Posible cierre" (30/60/90/fecha), bloqueando el guardado de la oferta ENTERA, no solo ese campo. Ya corregido — ver `guardarEnSP()` en "Fixes recientes": reintenta automáticamente sin el campo que Graph rechace y avisa con un toast cuál columna falta, para que nunca se pierda una cotización por un campo opcional sin crear.

### Feature: Posible Cierre (30/60/90 días) + Fecha de Seguimiento automática
- En "Nueva oferta", selector **"Posible cierre"** (barra fija, junto a Entrega/Voltaje): 30 / 60 / 90 días desde hoy, o "✏️ Elegir fecha..." para una fecha personalizada. Por defecto **"— Sin definir —"** — no se fuerza a poner fecha en cotizaciones exploratorias que no son un prospecto real.
- Solo se guarda **`FechaCierre`** en SharePoint (`guardarCotizacion()`). La **Fecha de Seguimiento nunca se guarda aparte** — siempre se calcula como `FechaCierre − 5 días` (`calcularFechaSeguimiento()`), para no tener 2 fuentes de verdad que se puedan desincronizar si alguien edita una fecha a mano en SharePoint.
- Al reabrir una oferta para editar/revisar (`editarOferta()`), la fecha de cierre se restaura siempre como "fecha personalizada" (no se intenta adivinar si originalmente fue 30/60/90) — sigue siendo editable.
- `cargarHistorial()` calcula `fechaCierre`/`fechaSeguimiento` al mapear cada registro — de ahí los consume tanto el Dashboard como la nueva pantalla de Seguimiento.

### Feature: pantalla de Seguimiento (por usuario, Día/Semana/Mes)
- Nuevo ítem de navegación **"Seguimiento"** (con badge rojo — cuenta vencidos + próximos 3 días) junto a Historial. Muestra SOLO las ofertas activas (no Cerrada-Ganada/Perdida) del usuario logueado (`asesorLabel(r.vendedor)===asesorLabel(currentUserName)`) que tengan Fecha de Seguimiento — nunca las de otro asesor.
- Usa `soloVigentes()` (la misma función del fix de revisiones) para que una oferta con -R2/-R3 no aparezca duplicada como 2 seguimientos.
- **"⚠ Vencidos"**: sección fija arriba, siempre visible sin importar qué rango se esté navegando (fecha de seguimiento ya pasada y la oferta sigue activa).
- 3 modos — Día y Semana como **lista organizada por fecha** (Semana muestra los 7 días, incluso vacíos, para ver el ritmo de la semana); Mes como **calendario en grilla** (7×6, clic en un día salta a la vista Día de esa fecha).
- Cada tarjeta muestra cliente, N° oferta, fecha de cierre, "Vencido hace N días" / "En N días", estado y total — clic salta al Historial con ese N° de oferta ya filtrado (`verOfertaDesdeSeguimiento()`).
- `cargarHistorial()` ahora también se llama en `showApp()` (antes solo cargaba al entrar a Historial) para que el badge esté listo desde el login, sin esperar a que el usuario abra Historial o Seguimiento primero.
- **Importante — qué significa "notificar" aquí:** esto es un aviso dentro de la app (badge + pantalla), no un correo/push real. Este cotizador es un sitio estático sin backend/cron — para un correo o notificación real automática haría falta un flujo externo (ej. Power Automate) que lea `FechaCierre` de la lista y envíe el aviso; esta app por sí sola no puede disparar eso.

### Feature: Firma digital del cliente en el PDF (canvas, sin servicio externo)
- El "PDF" en realidad es una página HTML (`generarPDF()`) que se abre en una ventana nueva; el botón "⬇ Guardar como PDF" solo llama a `window.print()` — el navegador es quien lo convierte a PDF. **No hay campos de formulario PDF reales** (no se usa jsPDF ni ninguna librería de PDF) — por eso la firma es un `<canvas>` dibujable, no un "campo" de Acrobat.
- En la sección de firmas (dentro de `condiciones`, lado del cliente): `<canvas id="firmaCanvas">` con un mensaje "✍️ Firme aquí" (clase `no-print`, no sale en el PDF impreso) superpuesto hasta que el cliente empieza a dibujar. El cliente firma con mouse o dedo (touch) directamente ahí — funciona en laptop, tablet o celular. Debajo sigue existiendo la línea en blanco con "Firma de aceptación · Fecha: ___" (se completa sola con la fecha real al firmar) — así si nadie usa el canvas, sigue sirviendo para firmar a mano sobre el papel impreso, exactamente como antes.
- Botones `no-print` (no salen en el PDF): **"🗑 Borrar firma"** (limpia el canvas) y **"💾 Guardar firma"** (aparece solo después de dibujar algo).
- "Guardar firma" vuelve a subir el HTML de esa misma ventana (ya con la firma dibujada) a SharePoint, **sobrescribiendo el mismo archivo** — llama a `window.opener.guardarHTMLEnSharePoint(numOferta, htmlFirmado, cliente, idSynergy)`, la misma función que ya usa el flujo normal de guardado. Como el nombre de archivo es determinístico (`nombreArchivoCot`), el link que ya quedó en el campo `LinkPDF` de SharePoint pasa a mostrar la versión firmada sin cambiar de URL. Si se cerró la ventana original del cotizador (`window.opener` nulo/cerrada), se avisa con un `alert()` en vez de fallar en silencio.
- **⚠️ Gotcha real que ya se corrigió aquí — no volver a introducirlo:** el HTML de `generarPDF()` inyecta un `<script>...</script>` completo (la lógica de dibujo) DENTRO de un string JS que vive en el `<script>` principal de este archivo. El parser HTML del navegador corta cualquier `<script>` en el primer `</script>` que encuentre **aunque esté dentro de un string** — así que el cierre de ese script inyectado se escribe `<\/script>` (con backslash) en el código fuente, nunca `</script>` liso. Si se edita este bloque y se quita el backslash, se rompe el `<script>` principal de todo el archivo (la app deja de cargar por completo). El resto de escapes de comillas ya usa `JSON.stringify()` para los valores interpolados (`d.numOferta`, `d.cliente`, `d.idSynergy`).
- Es una firma **visual/manuscrita digital**, con el mismo peso que firmar a mano sobre papel — no es una firma electrónica certificada (no hay verificación de identidad, sello de tiempo legal ni trazabilidad tipo DocuSign). Si en algún momento se necesita eso, es un proyecto aparte (servicio de firma electrónica pago + integración por API).

### Feature: enviar link de la oferta a un cliente externo para que firme
- **⚠️ CONFIRMADO 16-sep-2026 — el sitio de SharePoint (Comercial) tiene el uso compartido DESACTIVADO por completo.** `crearLinkAnonimoParaOferta()` llama a Graph `POST /drive/items/{id}/createLink` con `{type:'view',scope:'anonymous'}`; probado con el usuario real y Graph devuelve *"The operation failed because sharing has been disabled on this site."* — no es un error de código, es la configuración del sitio. Para habilitarlo: SharePoint Admin Center → Sites → sitio activo "Comercial" → Sharing, cambiar de "Only people in your organization" a algo que permita invitados/anónimos; o a nivel del sitio, Configuración → Permisos del sitio → "Cambiar cómo los miembros pueden compartir". Requiere un administrador de Microsoft 365 — no se puede tocar desde este repo. Mientras esto siga desactivado, el botón "📤 Link para el cliente" y `generarLinkClienteIdx()` del Historial van a fallar siempre con ese mismo mensaje — es esperado, no hay que "arreglarlo" en el código.
- Botón **"📤 Link para el cliente"** en la barra de la ventana del PDF (junto a "🔗 Ver en SharePoint", solo si `d.linkSharePoint` existe) y en el Historial (ícono ✈️ junto al de PDF, `generarLinkClienteIdx(id)`) — ambos llaman a `crearLinkAnonimoParaOferta(numOferta, cliente, idSynergy)` y muestran el link en un `prompt()` listo para copiar y enviar por correo/WhatsApp. **Sigue ahí para cuando se habilite** el compartir en SharePoint — no se quitó, solo no sirve todavía.
- **Alternativa que SÍ funciona ya, sin depender de esa configuración — botón "📧 Descargar para enviar por correo"** (`descargarHTMLParaCorreo()`, junto al de arriba, sin condición de `linkSharePoint`): descarga el HTML actual (con la firma si ya se dibujó) como archivo `.html` vía `Blob`+`URL.createObjectURL` — 100% del lado del navegador, cero llamadas a SharePoint. El vendedor adjunta ese archivo en un correo o lo manda por WhatsApp Web; el cliente lo abre localmente con doble clic (funciona perfecto como archivo local, JS y canvas no necesitan servidor) y firma igual que con el link.
- **El cliente, al abrir el HTML (por link o por archivo local), NO tiene `window.opener`** (no vino de `window.open()` de la app) — el script inyectado lo detecta (`hayOpener`) y en ese caso: oculta "📤 Link para el cliente" y "💾 Guardar firma" (ninguno de los 2 puede funcionar sin sesión de AYJ para escribir a SharePoint) y muestra en su lugar una nota fija: *"Después de firmar, use '⬇ Guardar como PDF' arriba y envíe ese archivo por correo o WhatsApp a AYJ Maquinaria"*. El cliente externo firma en el canvas y usa el botón de imprimir que YA existía — el PDF resultante se lo queda él y nos lo reenvía manualmente; no hay forma de que la firma vuelva sola a SharePoint sin que un usuario de AYJ (con sesión MSAL) la suba, porque este cotizador no tiene backend/credenciales de servicio para escribir sin login.
- Proceso recomendado end-to-end (con el uso compartido de SharePoint desactivado, es decir, HOY): (1) el vendedor genera la oferta como siempre; (2) clic en "📧 Descargar para enviar por correo" y adjunta ese `.html` al correo/WhatsApp; (3) el cliente lo abre localmente, firma en el canvas, y guarda/envía el PDF; (4) si se quiere que el registro oficial en SharePoint también quede con la firma, el vendedor puede reabrir esa misma oferta desde el Historial (ahí sí tiene `window.opener`) y usar "💾 Guardar firma" con el trazo del cliente, o simplemente archivar el PDF que devolvió el cliente en otro lugar. Si en el futuro se habilita compartir en SharePoint, el paso (2) puede cambiar a "📤 Link para el cliente" sin tocar código.

### Fixes recientes
- **17-sep-2026 — Guardar oferta fallaba por completo si SharePoint no tenía la columna FechaCierre (bug real reportado por el usuario).** `guardarEnSP()` dentro de `guardarCotizacion()` ahora detecta `"Field 'X' is not recognized"` en el error de Graph, reintenta automáticamente el POST sin ese campo, y avisa con un toast de advertencia cuál columna falta — la oferta se guarda igual, solo sin ese dato opcional. Genérico (no hardcodeado a "FechaCierre" — sirve para cualquier columna futura que aún no se haya creado en SharePoint). Ver sección de FechaCierre arriba. Aplicado también en V1.
- **16-sep-2026 — Botón "Descargar para enviar por correo" (fallback sin SharePoint).** Confirmado con el usuario real que el sitio Comercial tiene el uso compartido desactivado — el link anónimo falla siempre. Se agregó una alternativa que no depende de esa configuración. Ver sección arriba. Aplicado también en V1.
- **16-sep-2026 — Link de la oferta para que un cliente externo la firme.** Ver sección arriba — depende de que "Cualquier persona con el enlace" esté habilitado en SharePoint (confirmado desactivado por ahora). Aplicado también en V1.
- **16-sep-2026 — Firma digital del cliente en el PDF (canvas dibujable).** Ver sección arriba. Aplicado también en V1.
- **15-sep-2026 — Posible Cierre (30/60/90) + Fecha de Seguimiento automática + pantalla de Seguimiento por usuario.** Ver 2 secciones arriba — requiere crear la columna `FechaCierre` en SharePoint (ver aviso arriba). Aplicado también en V1.
- **15-sep-2026 — Agrupar revisiones (-R2, -R3...) en las estadísticas ("Solo vigentes").** Ver sección arriba. Aplicado también en V1.
- **12-sep-2026 — Asignar/reasignar Gerente de Cuenta a otra persona (no solo a uno mismo).** Ver sección de Gerente de Cuenta arriba. Aplicado también en V1.
- **12-sep-2026 — Ofertas del cliente + estadísticas/gráfico en su ficha.** Ver sección arriba. Aplicado también en V1.
- **12-sep-2026 — Gerente de Cuenta en Clientes, gráfico Ofertas x Mes, overlay de carga bloqueante.** Ver 3 secciones arriba. Aplicado también en V1.
- **12-sep-2026 — Jorge Pulido y Jorge Salamanca mezclados como un solo "Jorge" (incl. bug de permisos).** Ver sección arriba.
- **12-sep-2026 — Filtros Mes/Asesor + "ocultar incompletos" + 2 gráficos por asesor.** Ver secciones arriba.
- **12-sep-2026 — Gráfico Origen (Stock/Importación/Bajo Pedido).** Ver sección arriba.
- **12-sep-2026 — Tipología no se leía de SharePoint (nombre interno mal detectado).** Ver sección arriba — bug real, no falta de dato.
- **12-sep-2026 — Diagnóstico de "Sin tipología" (lista de ítems afectados al filtrar).** Ver sección arriba.
- **12-sep-2026 — 2 bugs reales en Inteligencia + 3 gráficos nuevos.** "Top Tipologías" mostraba "pedido" (fallback a `item.tipo`, disponibilidad, no tipología) y duplicaba el valor de ítems con cantidad>1 (multiplicaba `tarifaUSD` por `cantidad` de nuevo). "Ventas por Asesor" sumaba TODO lo cotizado en vez de solo lo ganado — por eso los números no cuadraban entre gráficos. Ver detalle arriba.
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
