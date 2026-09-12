---
name: AuditFile
description: Audita la salud de los datos de un sistema tipo ERP/POS (PRIMATE, MIXIT, mi-pos, GESIMP, o cualquier otro proyecto de datos del usuario) en 4 ejes -- Aprovechamiento, Integridad, Latencia y Veracidad -- y produce un informe con hallazgos concretos y priorizados. Usalo cuando el usuario pida "auditá los datos de X", "revisá la salud del dato", "cuanto estamos aprovechando el sistema", o invoque /AuditFile. Tambien aplica el protocolo de trazabilidad (seccion aparte) cada vez que se construye o revisa un reporte, panel o KPI en cualquier proyecto.
---

# AuditFile

Auditoría de datos en 4 ejes para sistemas tipo ERP/POS del usuario (PRIMATE, MIXIT, mi-pos-android6, GESIMP, NODO Cartera de Licencias, o cualquier otro con una base de datos real detrás). Diagnostica, no arregla solo -- entrega un informe con hallazgos concretos, cada uno con severidad y qué se ganaría si se corrige. Igual que cualquier trabajo sobre datos reales de este usuario: **solo lectura contra producción**, cualquier fix sigue el flujo test → OK del usuario → prod ya establecido en cada proyecto.

## Por qué existe

Nace de un incidente real en PRIMATE (10/09/2026): un stock duplicado por una carga manual + una recepción de factura para la misma mercadería pasó desapercibido durante más de un día, reportado recién cuando Marta (Encargada de Depósito) se hartó de corregirlo a mano. Un chequeo de integridad (Eje 2) sobre `SUM(MaterialLoteSaldo) = Material.StockActual` corriendo antes, o pedido bajo demanda, lo hubiera encontrado en minutos en vez de vía WhatsApp. Este skill sistematiza ese tipo de chequeo -- y los otros 3 ejes que el mismo caso dejó en evidencia (el "Aprovechamiento": ¿por qué la Recepción no le avisó a nadie del ingreso manual reciente del mismo material?).

## Alcance de una corrida

Se invoca con un alcance explícito -- no "auditá todo" sin acotar:

- Un proyecto completo (`/AuditFile PRIMATE`) -- razonable solo si el proyecto es chico o hay tiempo/presupuesto de tokens para una pasada larga.
- Un módulo o dominio (`/AuditFile PRIMATE — Materiales y Stock`, `/AuditFile mi-pos — Ventas y Caja`) -- el modo normal de uso.
- Una pregunta puntual (`/AuditFile ¿el StockActual de todos los materiales coincide con sus lotes?`) -- el modo más barato y el que conviene sugerir primero si el usuario no acotó.

Si el usuario no da alcance, preguntale cuál de los tres quiere antes de arrancar (una auditoría completa sin acotar puede recorrer decenas de tablas y salir muy cara en tokens para lo que en general hace falta).

## Paso 0 -- identificar el terreno

Antes de auditar nada, determiná:

1. **Stack de datos**: SQL Server vía APISQL/gateway (PRIMATE, GESIMP), Cloudflare D1 (mi-pos-android6, mipos-gateway), Supabase (varios clones/clientes), u otro. Cada uno cambia CÓMO se introspecciona el esquema y se corren las consultas, no el criterio de los 4 ejes.
2. **Cómo leés datos reales sin escribir nada**: en PRIMATE, el login `ClaudeAgent` + `SPClaudeEjecutarScript` permite `SELECT` libre contra test; contra PROD nunca hay acceso directo -- generá diagnósticos `_DIAGNOSTICO_*_PROD_v1.sql` de SOLO LECTURA para que el usuario los corra en SSMS y te pase el resultado, exactamente como en cualquier otro diagnóstico de este proyecto. En D1/Supabase, usá las credenciales/CLI que ya existan para ese proyecto (memoria del proyecto suele tenerlas) -- nunca inventes ni pidas credenciales nuevas.
3. **Qué tablas son "core"**: mirá el `CLAUDE.md` del proyecto (tabla de "Módulos implementados" si existe, como en PRIMATE) para priorizar las tablas que sostienen la operación real (ventas, stock, cobranzas) antes que catálogos periféricos.

## Los 4 ejes

Para cada uno: qué es, cómo se chequea en la práctica, y qué pinta tiene un hallazgo bien escrito (no "hay un problema de integridad" -- un número real, una tabla real, un impacto real).

### 1. Aprovechamiento del dato

¿Se está usando el sistema al máximo, o hay campos/módulos que existen pero nadie llena ni mira? Cada columna vacía es una de dos cosas: una feature muerta, o información que se podría estar aprovechando y no.

**Cómo chequear:**
- Para cada tabla core, `SELECT COUNT(*), COUNT(columna)` (o `SUM(CASE WHEN columna IS NULL OR columna = '' THEN 1 ELSE 0 END)`) por columna nullable/opcional -- calculá el % de filas donde está vacía.
- Marcá las que superan ~80-90% de vacío como candidatas a reportar.
- Para cada una, no te quedes en "está vacía" -- explicá qué HABILITARÍA llenarla: un filtro nuevo, un reporte que hoy no se puede armar, una segmentación de clientes/materiales, una alerta automática. Si no se te ocurre para qué serviría, decilo también (puede ser deuda de diseño, no oportunidad real).
- Mirá también tablas/módulos con actividad sospechosamente baja comparada con el volumen de negocio esperado (ej. una tabla de `Reserva` con 3 filas en un sistema que factura miles de líneas por mes) -- señal de que el módulo existe pero el equipo no lo adoptó.

**Ejemplo de hallazgo bien escrito:**
> `Material.Categoria` está vacío en el 94% de los 240 materiales activos de PRIMATE. Si se completara, `RptVentasProducto` podría agrupar por categoría sin tocar código (el campo ya está en el SELECT, solo no se usa como filtro) -- hoy Administración arma esa vista a mano en Excel una vez por mes.

### 2. Integridad del dato

¿Los números que el sistema muestra son *consistentes entre sí*? No es "¿es correcto el dato?" (eso es el Eje 4) sino "¿el sistema se contradice a sí mismo?".

**Cómo chequear:**
- **Invariantes contables/de negocio explícitos o implícitos**: totales de cabecera vs. suma de líneas, saldos acumulados vs. suma de movimientos, un campo cacheado (`StockActual`, `CostoPromedio`, un `Saldo` de cuenta corriente) vs. su fuente de verdad recalculada desde cero. El caso de hoy (`Material.StockActual` vs `SUM(MaterialLoteSaldo)`) es el ejemplo canónico -- cualquier sistema con un campo "cache" corre este mismo riesgo en cuanto dos caminos de código distintos pueden tocarlo.
- **Integridad referencial no reforzada por FK real**: filas hijas que apuntan a un padre que ya no existe o está anulado/inactivo (buscá FKs declaradas en el schema vs. las que en la práctica el código nunca garantiza).
- **Duplicados de clave natural** que no deberían coexistir (dos comprobantes con el mismo número y tipo activos a la vez, dos usuarios con el mismo documento).
- Para cada invariante que definas, la query es siempre la misma forma: calculá ambos lados por separado y contá cuántas filas difieren -- no asumas, contá.

**Ejemplo de hallazgo bien escrito:**
> `SUM(MaterialLoteSaldo.Cantidad)` no coincide con `Material.StockActual` en 5 de 240 materiales activos (barrido del 10/09/2026) -- diferencias de 1 a 11 unidades, sin relación con el incidente de facturas duplicadas del mismo día. Antigüedad no determinada, sugiere reconciliar aparte con bajo apuro.

### 3. Latencia del dato

¿La información llega a tiempo para decidir, o el sistema (o el proceso alrededor) hace que siempre se esté mirando el pasado?

**Cómo chequear:**
- Compará la fecha/hora en que un evento de negocio *ocurrió* contra la fecha/hora en que quedó *registrado* en el sistema (si hay ambos campos) -- una brecha sistemática de horas/días es proceso batch disfrazado de tiempo real.
- Para reportes/paneles que dependen de un campo pre-calculado (no de una consulta en vivo), verificá qué lo actualiza y con qué frecuencia -- si depende de un cron/proceso manual que no corrió, el panel puede estar mostrando ayer sin que nadie lo note.
- Preguntale al usuario (o inferí del proceso descrito en el `CLAUDE.md`/memoria del proyecto) si hay pasos manuales entre "pasó en la realidad" y "está en el sistema" (ej. "se carga la planilla de ventas del día siguiente por la mañana") -- eso es latencia de PROCESO, no de código, y vale la pena nombrarlo igual aunque no sea arreglable con una query.

**Ejemplo de hallazgo bien escrito:**
> El Kardex de PRIMATE muestra `CostoPromedio` recalculado solo al confirmar una Recepción -- entre una recepción y la siguiente (a veces días), cualquier consumo de OT imputa a un costo promedio que ya no refleja compras más recientes sin confirmar todavía. No es un bug: es una latencia de costeo inherente al diseño, vale la pena que Administración lo sepa al leer el reporte de rentabilidad.

### 4. Veracidad del dato

Aunque el dato sea internamente consistente (Eje 2) y esté a tiempo (Eje 3), ¿es *razonable*? ¿Podría ser cierto, o es obviamente un error de carga/cálculo?

**Cómo chequear:**
- **Rangos imposibles o absurdos para el dominio de negocio**: precios en 0 o negativos donde no corresponde, cantidades órdenes de magnitud fuera de lo típico para ese producto/cliente, fechas futuras o anteriores a la existencia del negocio, porcentajes fuera de 0-100 donde deberían estarlo.
- **Plausibilidad cruzada**: dos campos que deberían moverse juntos y no lo hacen (un `CostoPromedio` muy distinto de `PrecioCosto` sin ningún movimiento que lo explique), un total que no es ni remotamente `cantidad × precio unitario`.
- Para montos con volumen suficiente (cientos+ de filas), la Ley de Benford puede señalar manipulación o generación artificial de datos -- técnica avanzada, no la actives por default; mencionala como opción si el usuario sospecha fraude en vez de solo error de carga (ahí conviene derivar directamente a Cons.Fraude en vez de este skill).

**Ejemplo de hallazgo bien escrito:**
> 3 líneas de `ComprobanteLinea` en PRIMATE tienen `PrecioUnitario = 0` con `Cantidad > 0` sobre materiales que en cualquier otra compra cuestan miles de Gs -- no es imposible (podría ser una donación/canje) pero amerita confirmar con Compras antes de asumir que el costeo de esas 3 OT es real.

## Formato del informe

Un hallazgo por bloque, ordenados por severidad (ALTA/MEDIA/BAJA, mismo criterio que cualquier otro informe de este usuario: ALTA = afecta una decisión de negocio real hoy o ya causó daño; MEDIA = riesgo latente o oportunidad de aprovechamiento clara; BAJA = cosmético o de bajo impacto):

```
[EJE] [SEVERIDAD] Título de una línea
Qué se encontró (con números reales, no genérico).
Por qué importa / qué se ganaría arreglándolo o llenándolo.
(Si aplica) qué se necesitaría para confirmarlo o corregirlo -- y si eso requiere
tocar PROD, decilo explícito y no lo hagas sin el OK del usuario.
```

Cerrá siempre con un resumen de 1-2 líneas: cuántos hallazgos por eje, y cuál es el más urgente de los cuatro si tuviera que elegir uno.

No fuerces encontrar algo en un eje si no hay nada real -- "Eje 3 (Latencia): sin hallazgos, los datos revisados se registran en tiempo real" es un resultado válido y esperable, igual que en cualquier barrido de este usuario (ver memoria de PRIMATE sobre auditorías: confirmar "no-hallazgos" con evidencia es tan valioso como encontrar un bug).

## Protocolo de trazabilidad (aplica siempre, no solo durante una auditoría)

Regla permanente para **cualquier reporte, panel o KPI que se construya o se toque** en cualquier proyecto de este usuario, no solo cuando corre este skill -- traela a colación activamente cuando estés escribiendo o revisando un `Rpt*.js`/dashboard/panel de KPIs:

1. **Sección de trazabilidad visible en la pantalla misma** (no en un documento aparte): un ícono o texto tipo "¿Cómo se calcula esto?" que al abrirse explica en lenguaje llano la fórmula/fuente real detrás del número que se está mirando -- de dónde sale cada término, no un mensaje genérico tipo "suma de ventas".
2. **Detalle de comprobantes**: en otra parte de la misma pantalla (o un drill-down a un click), el listado completo de los documentos/movimientos reales que componen ese número -- para que alguien pueda verificar a mano si quiere, documento por documento.

Si el proyecto es PRIMATE, esto extiende (no reemplaza) el estándar ya existente en `CLAUDE.md` bajo "Estandar obligatorio — Reportes" (Exportar Excel, Vista previa/Impresión, Días vencidos) -- proponele al usuario sumarlo ahí como punto 4 formal la primera vez que toques un reporte existente que no lo tenga, en vez de asumir que ya aplica.

Al auditar un reporte/panel EXISTENTE bajo este skill, chequear si cumple este protocolo es en sí mismo un hallazgo de Eje 1 (Aprovechamiento) o Eje 4 (Veracidad) según el caso -- "este KPI no muestra cómo se calcula" es exactamente el tipo de hallazgo que este skill existe para sacar a la luz.

## Qué NO hacer

- No corras ningún `UPDATE`/`DELETE`/fix contra datos reales como parte de una auditoría -- eso es un paso aparte, después, con el OK explícito del usuario (mismo flujo que cualquier otro trabajo de datos de este usuario).
- No inventes hallazgos para "llenar" los 4 ejes -- un eje sin nada real es un resultado válido.
- No repitas una auditoría completa de un proyecto grande sin que el usuario la pida de nuevo -- es cara en tokens; para seguimiento, preguntá primero si conviene acotar al área que más preocupa.
