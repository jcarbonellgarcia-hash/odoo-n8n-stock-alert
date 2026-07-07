# Bitácora técnica — Stock Alert (Tornillo M8 / Tuerca M6)

Integración n8n + Odoo. Workflow que monitoriza stock de dos productos, compara contra mínimo configurado, y alerta por Telegram solo en el cambio de estado (OK → bajo mínimo), evitando fatiga de notificación.

Este documento recoge cada bloqueo real encontrado durante el desarrollo, el diagnóstico erróneo inicial cuando lo hubo, y la solución final aplicada. Se documenta también la deuda técnica conocida y no resuelta, de forma transparente.

## Arquitectura resumida

- Trigger: `CheckEveryHour` (Cron).
- Dos ramas paralelas de consulta HTTP a la API de Odoo (Tornillo M8 / Tuerca M6).
- Transformación de datos (`Split Out`, `Edit Fields`, `Summarize`).
- Comparación contra mínimo configurado (`Filter`).
- Deduplicación de alertas vía tabla de estado (`Data Table`: `stock_alert_state`).
- Notificación por Telegram solo en cambio de estado.
- Reset de estado cuando el stock vuelve a nivel normal.

## Bloqueos y resolución

### 1. Campo `many2one` sin aplanar

`stock.quant.product_id` no es un ID escalar: es un array `[id, "display_name"]`. El nodo `Merge` (Combine, Matching Fields) comparaba ese array crudo contra un ID de `product.product` y nunca encontraba coincidencia, devolviendo "No output data" sin explicación.

Diagnóstico erróneo inicial: se sospechó un problema de nombre de campo en el desplegable "Fields to Match".

Diagnóstico real: había que extraer el escalar con un nodo `Set` (`{{$json.product_id[0]}}`) antes de cualquier `Summarize` o `Merge`.

Complicaciones adicionales durante la corrección:
- Primer intento de `Set` probado de forma aislada mostró datos de una ejecución anterior cacheada, no del flujo real en cadena.
- El toggle "Include Other Input Fields" del nodo `Set` parecía activo visualmente pero evaluaba a `false` fijo, descartando todos los campos salvo el mapeado a mano. Solución: mapeo explícito de cada campo necesario (`productId`, `quantity`), sin depender del toggle.

### 2. Modelo equivocado en la consulta de mínimos

La rama de mínimos consultaba `product.template` con dominio `is_storable=true`. El ID de `product.template` coincidía por casualidad con el `productId` de `stock.quant` — alineación accidental, no garantizada, ya que el catálogo no tiene variantes de producto. Se documenta como deuda técnica (ver más abajo), no se resuelve en este ciclo.

### 3. Campo personalizado no-stored

Al filtrar `reordering_min_qty > 0` directamente en el dominio de Odoo (`search_read`), error: *"Field has no SQL representation because it is not stored"*. El campo se creó vía Studio sin `store=True`.

Decisión: no modificar la definición del campo en el modelo de Odoo sin autorización del cliente. Se resolvió filtrando en la capa de integración, con un nodo `Filter` en n8n.

### 4. Selector de campos con schema cacheado

Al montar el `Merge` final, el selector de "Fields to Match" mostraba el schema de una ejecución anterior a la corrección del `Set`, sin ofrecer `productId` como opción disponible. Solución: ejecutar el workflow completo antes de reabrir el nodo `Merge`, no solo el nodo aislado.

### 5. Diseño de deduplicación de alertas

Sin control de estado, el flujo mandaba la misma alerta cada hora mientras el producto siguiera bajo mínimo (alert fatigue). Diseño: tabla de estado `stock_alert_state` (Data Table de n8n), columnas `productId`, `alerted`, `lastAlertDate`.

Incidencias al crear la tabla:
- Primer intento generó una tabla vacía sin columnas.
- Error de uso: se escribieron nombres de columna como si fueran filas de datos, en vez de usar el botón "Add Column".
- La columna `lastAlertDate` fue rechazada inicialmente por "invalid column name" — resultó ser un error de tipeo, no una restricción real del sistema.

### 6. Pérdida silenciosa de items sin coincidencia

Con "Always Output Data" activado en `Get Row(s)`, se asumió que el nodo rellenaría huecos para items sin match. Esto es falso: si de 2 items de entrada solo 1 tiene coincidencia, el nodo devuelve únicamente ese 1 — el otro desaparece sin error. "Always Output Data" solo actúa si el resultado global sería cero, no por item individual dentro de un batch. Este fue el bloqueo más difícil de diagnosticar por no producir ningún error visible.

### 7. Rediseño de la arquitectura de deduplicación

Como paliativo al bloqueo 6, se probó un segundo `If` con expresión ternaria (`{{$json.alerted === true ? "SI" : "NO"}}`), evitando depender de los operadores booleanos del desplegable ("is not true" no existe; "is false" fallaba con valores `undefined`). Esto resolvió el caso "producto ya alertado", pero no el caso "producto nunca alertado", porque `Get Row(s)` seguía perdiendo esos items.

Solución final: sustituir el segundo `If` por un nodo `Merge` en modo *Combine* + *Keep Non-Matches* + *Output Data From: Input 1*, comparando el conjunto completo de productos bajo mínimo contra el conjunto completo de productos ya alertados. Para esto, `Get Row(s)` se configuró con "Return All" + "Execute Once", trayendo la tabla completa en una sola pasada en vez de una consulta por item.

Incidencia adicional: con "Fields To Match Have Different Names" activado pero ambos campos llamados igual (`productId`), el matching no resolvía correctamente. Desactivar ese toggle lo corrigió.

### 8. Reset de estado al recuperarse el stock

Faltaba un `Upsert row(s)` en la rama `false` del primer `If` (stock por encima de mínimo), para marcar `alerted: false`.

Incidencias al montarlo:
- Error de expresión: se arrastró el campo desde el panel en vez de teclear la expresión, dejando texto literal (`productId = {{$json.productId}}`) en vez de la expresión evaluada.
- Se intentó mapear `lastAlertDate` con un valor por defecto `false` en una columna tipada `dateTime`, rechazado por el sistema. Solución: no mapear ese campo en el reset, dejarlo `null`.

## Cierre y verificación

Flujo probado de extremo a extremo. Evidencia: captura del canvas mostrando el disparo real (item viajando por `Merge` → `Telegram`/`Upsert`), y captura de Telegram con dos alertas legítimas de productos distintos, sin duplicados.

## Deuda técnica conocida (no resuelta, documentada por transparencia)

- `reordering_min_qty` no tiene `store=True` en Odoo. Mitigado con `Filter` en n8n en vez de resolverlo en el modelo — pendiente de autorización del cliente para corregirlo en origen.
- Alineación entre IDs de `product.template` y `product.product` es accidental: funciona solo porque el catálogo actual no tiene variantes de producto. Se rompería si se añaden variantes.
- Sin gestión de error si el `HTTP Request` a Odoo devuelve un código 5xx — el sistema fallaría en silencio.
