---
name: dashboard-3eriza
description: "Dashboards analíticos XSell (fondo oscuro #1A1025, naranja #FF6B13, púrpura #5D17A0, logo XSell + P&C), responsive y optimizados para móvil. MySQL (intranetpbx 95+ BDs, CASTI) → Supabase → Edge Functions → HTML con Chart.js. Plantillas: Inbound, Outbound, Encuestas, Perfilamiento, Reclamos. SPD/HPD configurable. Exploración post-dashboard + menú analista. Activar para: dashboards, KPIs, predicciones, estrategias, SQL, queries, Supabase, sync, reportes, correo resumen, explorar datos, cruzar tablas, segmentar, buscar tablas MySQL, dashboard responsive, optimizar dashboard para móvil. También: métricas, tendencias, forecast, cuadros de mando, bases de datos, datos tabulares, visualizaciones, reclamos, libro de reclamaciones."
---

# Dashboard Analítico XSell — Planeamiento y Control

Este skill crea dashboards analíticos de alto nivel con la identidad corporativa de XSell. Cada dashboard incluye análisis de datos, predicción de escenarios, conclusiones accionables y estrategias de mejora. Maneja SQL avanzado (normalización, scripts, queries optimizados) e integración completa con dos servidores MySQL y Supabase como capa de datos. Incluye plantillas por tipo de servicio (Inbound, Outbound, Encuestas, Perfilamiento, Reclamos), flujo obligatorio de descubrimiento de datos, y exploración post-dashboard.

## Principios Fundamentales

Cada dashboard que produzcas debe cumplir tres funciones simultáneamente:

1. **Informar**: Mostrar el estado actual de los datos con claridad absoluta
2. **Predecir**: Proyectar escenarios (optimista, base, pesimista) usando los datos disponibles
3. **Recomendar**: Entregar conclusiones y estrategias concretas de mejora

No construyas dashboards meramente descriptivos. El usuario espera inteligencia analítica: patrones, anomalías, proyecciones y un plan de acción.

---

## Infraestructura y Conexiones

### Servidores MySQL

| Servidor | Host | Puerto | Usuario | Uso |
|----------|------|--------|---------|-----|
| **Principal** | `intranetpbx.net.pe` | 33306 | `kevin.principe` | 95+ BDs, tablas de gestión/tipificación por cliente (VOX_TRES, etc.) |
| **CASTI** | `66.206.10.130` | 3306 | `franz.palma` | BD `bd_casti` — asistencias, trabajadores, servicios, sedes |

### Supabase

| Recurso | Valor |
|---------|-------|
| Proyecto | `ejxpojzuovtsplviafbw` |
| Organización | TFIN (`pqjexgumjugaxsplllhn`) |
| Destino dashboards | Portal XSell centralizado (los analistas suben el HTML directamente) |

### Edge Functions Desplegadas

| Función | Versión | Propósito |
|---------|---------|-----------|
| `dashboard-query` | v8 | API principal: search, describe, query, sync_execute, sync_list, sync_delete, sync_auto_config. v8: `dateStrings:true` — fechas MySQL como TEXT (fix 0000-00-00) |
| `dashboard-data` | v2 | Proxy para dashboards HTML. Sin JWT, CORS abierto. Acciones: `tables`, `columns`, `download`, `query`, `values`. Respuesta siempre `{d: {data: [...]}}` |
| `deploy-dashboard` | v6 | Publica dashboards HTML en Netlify. Incluye `_headers` con CSP + HSTS + X-Frame-Options automáticamente |
| `auto-sync` | v2 | Cron cada hora, refresca tablas marcadas con `auto_sync: true` en `sync_registry` |
| `visor-auth` | v6 | Autenticación del VisorSQL. Password en `VISOR_PASSWORD` env var. Sesiones 30 min |
| `foes-hc-proxy` | v8+ | HC FOES desde Google Sheet. Cuenta `Perfil="Ejecutivo de campo"` AND (`D_Activos="ACTIVO"` OR `F_CESE` vacío) |
| `sync-asistencias-casti` | v6 | Sync diario 15:00 UTC. MySQL2 con `dateStrings:true`. Batches 5,000 filas |
| `excel-lookup` | v1 | Búsqueda cruzada desde Excel. Recibe registros del Excel, busca coincidencias (exactas + parciales) en Supabase + CASTI + Principal. Ver sección "Flujo B" |
| `mysql-connector` | v3 | Discover y query sobre MySQL CASTI (servidor legacy) |
| `mysql-describe` | v2 | Utilidad: describe tablas MySQL y guarda resultado en Supabase (usa pg_net) |

### VisorSQL — Explorador de Bases de Datos

**URL**: `https://visor-sql-3eriza.netlify.app` | **Contraseña**: `xsell-pc-2026`
**Repo GitHub**: `franzfernandops-max/visor-sql-3eriza` (auto-deploy en Netlify al push a `main`)
**Versión actual**: v4 — identidad XSell, logo oficial PNG, login con contraseña SHA-256

El VisorSQL es la herramienta centralizada para que los analistas descubran tablas MySQL y generen comandos de sync. Es el **punto de entrada obligatorio** antes de crear cualquier dashboard.

**Features v4:** Multi-servidor simultáneo (PRINC + CASTI), chips diferenciados `SUPA`/`PRINC`/`CASTI`, selección múltiple con comandos en bloque, pestaña "Tablas Supabase" con conteo en tiempo real, sesión en `sessionStorage`.

**Flujo del analista:** Buscar → Describir → Seleccionar ✅ → Ver comandos → Copiar → Pegar en Claude.

**REGLA:** Claude SIEMPRE entrega el VisorSQL primero. NUNCA crear un HTML explorador alternativo.

### Tabla `sync_registry` (registro central de sincronizaciones)

Columnas: `id`, `mysql_database`, `mysql_table`, `supabase_table`, `columnas` (array), `filtro_where`, `total_filas`, `analista`, `estado` (activo/eliminado), `creado_en`, `ultimo_sync`, `sync_count`, `auto_sync` (boolean), `frecuencia` (cada_hora/cada_6h/diario/semanal/off), `hora_sync`, `dia_sync`, `ultimo_auto_sync`, `errores_consecutivos`, `dashboard_url`.

---

## Flujo Obligatorio de Descubrimiento de Datos

### REGLA CRÍTICA: NUNCA asumir base de datos ni estructura

Cada cliente XSell usa un CRM distinto (Genesys, Avaya, Five9, CRM propio). Las tablas, columnas y estados varían completamente. El flujo siempre es:

**Paso 1 — Entregar el VisorSQL (OBLIGATORIO SIEMPRE PRIMERO)**
👉 **https://visor-sql-3eriza.netlify.app** (contraseña: `xsell-pc-2026`)

Decirle al analista: "Abre el VisorSQL, busca las tablas del cliente, haz clic en Describir y copia el comando sync para pasármelo." Claude espera el comando sync. **NUNCA continuar sin él. NUNCA crear un explorador HTML alternativo.**

**Si el analista no sabe usar el VisorSQL o tiene problemas**, guiarlo paso a paso:
1. Abrir `https://visor-sql-3eriza.netlify.app`
2. Ingresar contraseña: `xsell-pc-2026`
3. Escribir el nombre del cliente en el buscador (ej: "SCANIA", "LAIVE")
4. Hacer clic en **Describir** junto a la tabla que aparezca
5. Copiar el bloque de comandos que aparece en pantalla y pegarlo en este chat

**Paso 2 — Buscar en TODAS las BDs con `search`** (solo si el analista no tiene acceso al VisorSQL)
Usar la acción `search` de `dashboard-query` para buscar la palabra clave en los nombres de tablas y BDs de todo el servidor MySQL (95+ BDs).

```
dashboard-query?action=search&q=SCANIA&key=3eriza-pc-2026
```

**Paso 3 — Identificar tablas en orden de prioridad**
1. **Gestión/Tipificación** (LISTA_DET_*, DET_*, gestiones): Resultados de llamadas, tipificaciones, contactabilidad
2. **Tiempos/Estados** (TIEMPO_*, estados, login): Tiempos por estado del asesor, productividad
3. **Asistencia**: Asistencia del personal al servicio
4. **Chats/WSP**: Detalle de interacciones por WhatsApp u otros canales

**Paso 4 — Describe cada tabla encontrada**
```
dashboard-query?action=describe&database=VOX_TRES&table=LISTA_DET_SCANIAARG&key=3eriza-pc-2026
```

**Paso 5 — Elegir columnas y sincronizar**

### REGLA CRÍTICA: Límite de datos a sincronizar — máximo año en curso o últimos 3 meses

**NUNCA sincronizar data histórica de años anteriores sin restricción.** Las tablas MySQL pueden tener millones de filas históricas que causan timeouts en el sync, errores en los chunks de carga a Supabase y dashboards lentos.

Regla por defecto (aplicar siempre salvo que el analista pida explícitamente otro período):
- **Tablas de gestión/tipificación**: solo el **año en curso** (`WHERE YEAR(col_fecha) = YEAR(CURDATE())`)
- **Si la tabla no tiene data del año en curso**: últimos 3 meses (`WHERE col_fecha >= DATE_SUB(CURDATE(), INTERVAL 3 MONTH)`)
- **Si el analista necesita histórico**: máximo 12 meses, nunca más. Advertir del tiempo estimado.

⚠️ **NUNCA asumir que la columna de fecha se llama `fecha`.** Cada CRM la nombra distinto: `FECHA_GESTION`, `FEC_LLAMADA`, `FECHA_HORA`, `created_at`, etc. Identificar el nombre real en el `describe` (paso 4) ANTES de construir el `WHERE`. Si hay varias columnas de fecha, preguntar al analista cuál usar para el filtro de período. Si la tabla no tiene ninguna columna de fecha, advertir que no se puede aplicar filtro temporal y preguntar cómo proceder (sincronizar completa con límite de filas, o cancelar).

### REGLA CRÍTICA: Verificar si la tabla ya existe en Supabase antes de crearla

```sql
-- Verificar existencia ANTES de ejecutar el DDL (robusto, no afectado por RLS de catálogo)
SELECT EXISTS (
  SELECT 1 FROM information_schema.tables
  WHERE table_schema = 'public' AND table_name = 'nombre_tabla'
) AS existe;
```
- Si devuelve `false` → crear normalmente con `execute_sql` + DDL
- Si devuelve `true` → **PREGUNTAR** al analista: "La tabla ya existe con X filas. ¿Quieres reemplazarla (DROP + CREATE) o agregar datos nuevos (INSERT ignorando duplicados)?"

### Polling del sync — máximo 10 intentos

Después de disparar el sync con `pg_net.http_get`, verificar la respuesta así:

```sql
-- Repetir hasta obtener status_code, máximo 10 veces con ~5 seg entre intentos
SELECT id, status_code, content::text
FROM net._http_response WHERE id = [request_id];
-- null = procesando | 200 = éxito | 4xx/5xx = error
```

Si después de 10 polls sigue en null, decir al analista: "El sync está procesando en segundo plano (tabla grande). Verificar en 5 minutos con `SELECT COUNT(*) FROM tabla`."

### Validación post-sync (obligatoria)

Después de confirmar status 200, ejecutar siempre:

```sql
-- 1. Conteo total
SELECT COUNT(*) FROM nombre_tabla;

-- 2. Rango de fechas y nulos críticos
-- Reemplazar col_fecha por el nombre real de la columna de fecha identificada en el describe
SELECT
  MIN(col_fecha) AS fecha_min,
  MAX(col_fecha) AS fecha_max,
  COUNT(*) FILTER (WHERE col_fecha IS NULL) AS fechas_nulas
FROM nombre_tabla;
```

Si `fechas_nulas` es alto (>5% del total), alertar al analista antes de construir el dashboard.

Los estados del CRM/ACD varían por cliente. Antes de crear cualquier tabla de tiempos:
1. **Descubrir** qué CRM/ACD usa el cliente
2. **Listar** todos los estados/códigos disponibles del sistema origen
3. **Mapear** cada estado a categorías: Productivo (en llamada, ACW), Disponible (ready, idle), Pausa (break, almuerzo), No productivo (capacitación, reunión)
4. **Preguntar** al analista si el mapeo es correcto antes de crear tablas

---

## Flujo B — Búsqueda desde Archivo Excel

Este flujo se activa cuando el usuario sube un archivo Excel en el chat. El objetivo es identificar qué tablas (Supabase, CASTI, servidor Principal) contienen datos que coincidan con los del Excel, y opcionalmente construir un dashboard con ellas.

### Cuándo activar

El usuario adjunta un `.xlsx`/`.xls`, o dice "tengo un Excel con...", "busca estos registros", "¿en qué tabla están estos datos?"

### Paso 1 — Leer y mostrar resumen

```python
import openpyxl, json
wb = openpyxl.load_workbook('/mnt/user-data/uploads/archivo.xlsx')
ws = wb.active
headers = [cell.value for cell in ws[1]]
rows = []
for row in list(ws.iter_rows(min_row=2, values_only=True))[:500]:
    rows.append({str(headers[i]): str(v or '') for i, v in enumerate(row)})
print(f"Filas: {len(rows)} | Columnas: {headers}")
```

Mostrar al usuario: N filas, columnas detectadas con tipo inferido (DNI, teléfono, nombre, email, texto libre), muestra de 3 filas.

### Paso 2 — Disparar `excel-lookup` via pg_net

```sql
-- pg_net.http_post usa parámetros POSICIONALES: (url, body, headers) — NO usar named params (url:=, body:=)
SELECT net.http_post(
  'https://ejxpojzuovtsplviafbw.supabase.co/functions/v1/excel-lookup',
  jsonb_build_object(
    'key',     '3eriza-pc-2026',
    'records', '[...array JSON del Excel...]'::jsonb,
    'sources', '["supabase","casti","principal"]'::jsonb
  ),
  '{}'::jsonb,
  '{"Content-Type":"application/json"}'::jsonb
) AS request_id;
-- Firma: net.http_post(url, body, params, headers, timeout_ms)
-- Luego: SELECT id, status_code, content::text FROM net._http_response WHERE id = [request_id];
```

**Si el JSON supera ~4,000 chars:** dividir en lotes de 100 filas y combinar resultados.

### Paso 3 — Presentar resultados

La EF devuelve por tabla: `source`, `db`, `table`, `total_rows`, `column_matches[]`, `best_match_pct`, `has_multi_col_match`.

Formato de presentación en el chat:
```
📊 Resultados — 120 registros del Excel buscados en 3 fuentes

✅ EXACTO — lista_det_scaniaarg (Supabase · VOX_TRES)
   87/120 registros (72%) · columna `cuit` · multi-columna: Sí
   [tabla con 3 filas de muestra]

⚠️ PARCIAL — asistencias_casti (CASTI · bd_casti)
   34 coincidencias parciales · columna `nombre_trabajador`
   [tabla con 3 filas de muestra]
```

### Paso 4 — Preguntar qué hacer

> "¿Con qué tabla(s) creamos el dashboard? Puedes elegir una o combinar varias."

- **"Crea el dashboard con [tabla X]"** → Flujo A desde sync
- **"Combina [tabla X] + [tabla Y]"** → sincronizar ambas → dashboard multi-tabla
- **"Dame más detalle de [tabla X]"** → describe columnas y origen MySQL
- **"Solo necesitaba saber dónde están"** → terminar

### Regla de oro: un dashboard puede tener múltiples tablas

NUNCA asumir una sola tabla. Siempre ofrecer combinar. Un dashboard típico cruza: gestiones + asistencias + tiempos/estados.

### Límites técnicos

- Máximo 500 filas por búsqueda. Si el Excel tiene más, usar las primeras 500 e informar.
- Búsqueda en 95+ BDs del Principal puede tardar 15-30 seg — avisar al usuario.
- Match exacto (IN) tiene prioridad sobre parcial (LIKE). Siempre distinguirlos claramente.
- `has_multi_col_match: true` = alta confianza de que es la tabla correcta.

---

## Plantillas por Tipo de Servicio

### Inbound (Atención Entrante)
**KPIs**: SLA, Abandono, **TMO**, FCR, Llamadas Atendidas, Llamadas/Hora
**Vista Asesor**: Tiempos por estado + KPIs individuales + **TMO individual**. NO lleva SPD/HPD.
**Gráficos**: Volumen por franja horaria, tendencia SLA, distribución de motivos, abandono por hora pico

### Outbound (Gestión Saliente — Cobranzas, Televentas, Retención)
**KPIs**: Contactabilidad, RPC, Conversión, SPD, **TMO**
**Vista Asesor**: Tiempos por estado + KPIs individuales + **SPD** como KPI estrella + **TMO individual**
**Gráficos**: Embudo de contactabilidad, tendencia conversión, ranking asesores por SPD

### Encuestas
**KPIs**: Encuestas Completadas, Tasa de Respuesta, Completitud, HPD, **TMO**
**Vista Asesor**: Tiempos por estado + KPIs individuales + **HPD** como KPI estrella + **TMO individual**
**Gráficos**: Avance vs meta diaria, distribución de respuestas, completitud por asesor

### Perfilamiento de Leads
**KPIs**: Leads Perfilados, Penetración, Contactabilidad, HPD, **TMO**
**Vista Asesor**: Tiempos por estado + KPIs individuales + **HPD** como KPI estrella + **TMO individual**
**Gráficos**: Embudo de perfilamiento, distribución por score (Hot/Warm/Cold), avance de base

### Reclamos (Atención de Reclamos / Libro de Reclamaciones)
**KPIs**: Total Reclamos, Tasa de Atención (ANSWER/total), Contacto Efectivo (%), Abandono (%), HPD, **TMO**
**Vista Asesor**: Tiempos por estado + KPIs individuales + **HPD** como KPI estrella + **TMO individual**
**Canales**: Depende del cliente — puede ser solo telefónico, solo chat/WSP, o **multicanal (teléfono + chat)**. Si es multicanal, el dashboard DEBE incluir una sección por canal con KPIs separados y una comparativa cruzada.
**Gráficos obligatorios**:
- Tendencia mensual de volumen (atendidas vs no atendidas)
- Distribución por franja horaria (detectar picos de abandono)
- Distribución por categoría/tipificación de contacto (efectivo, no efectivo, rellamar, descartado)
- Top motivos/subcategorías de reclamo
- Ranking de asesores por volumen y contactabilidad
**Gráficos multicanal** (si aplica):
- Tendencia de chats WSP por mes
- Tipificación de chats (NIVEL_1 + NIVEL_2)
- Comparativa de contacto efectivo: teléfono vs chat
**Análisis especial**: Buscar concentración excesiva en pocos asesores (riesgo operativo). Si un asesor gestiona >50% del volumen, señalar como hallazgo crítico.

### Fórmula TMO (TODOS los tipos de servicio)

```
TMO = Tiempo en llamada (TALK)
```

**Reglas:**
- TMO es **solo el tiempo de conversación** (campo TALK o equivalente del CRM). NO incluye ACW ni post-atención
- Se muestra como **tarjeta KPI obligatoria** en los 5 tipos de servicio (Inbound, Outbound, Encuestas, Perfilamiento, Reclamos)
- Formato de visualización: `MM:SS` para promedios (ej: "04:32") o en segundos si el dato viene así
- En la Vista Asesor, mostrar **TMO individual** como tarjeta KPI por asesor
- Si el campo TALK viene como texto o en formato no numérico, **limpiar/convertir a segundos** antes de calcular promedios
- NUNCA asumir que TMO = TALK + ACW. En XSell, TMO = solo TALK

### Fórmula SPD/HPD (Outbound, Encuestas, Perfilamiento, Reclamos — NO Inbound)

```
SPD / HPD = Hits del día ÷ (Horas logueado ÷ Horas turno estándar)
```

**Reglas:**
- Se mide solo al **cierre del día**, NO en tiempo real
- La meta es **configurable por campaña/cliente** — NUNCA hardcodear una meta fija
- El denominador normaliza: un asesor con medio turno es comparable con uno de turno completo

**Ejemplos:**
- 3 ventas · 6h logueado · Turno 8h → SPD = 3 ÷ 0.75 = **4.0**
- 20 encuestas · 8h logueado · Turno 8h → HPD = 20 ÷ 1.0 = **20.0**
- 8 leads · 4h logueado · Turno 8h → HPD = 8 ÷ 0.5 = **16.0**

---

## Identidad Visual Corporativa XSell

### Paleta de Colores (NO NEGOCIABLE)

**Colores Principales** (Manual de Identidad XSell):

| Rol | Nombre | Hex | RGB | Uso en dashboards |
|-----|--------|-----|-----|-------------------|
| **Naranja primario** | Naranja XSell | `#FF6B13` | 255,107,19 | CTAs, barras principales, variaciones +, borde resumen ejecutivo |
| **Púrpura primario** | Morado XSell | `#5D17A0` | 93,23,160 | Gradientes de área, embudo, tracks de gauge, badges |

**Colores Secundarios** (Manual de Identidad XSell):

| Rol | Nombre | Hex | RGB | Uso en dashboards |
|-----|--------|-----|-----|-------------------|
| **Amarillo** | Amarillo | `#F6A500` | 246,165,0 | Acento terciario, estrellas, alertas medias |
| **Magenta** | Magenta | `#8D229E` | 141,34,158 | Acento adicional en gráficos multicanal |
| **Gris claro** | Gris Claro | `#EEEEEE` | 238,238,238 | Fondos en versiones light |
| **Gris oscuro** | Gris Oscuro | `#BFBFBF` | 191,191,191 | Texto secundario en versiones light |

**Colores de sistema P&C** (derivados de la paleta XSell para dashboards analíticos):

| Rol | Hex | Uso |
|-----|-----|-----|
| **Fondo página** | `#1A1025` | Fondo de toda la página — SIEMPRE este color |
| **Fondo tarjetas** | `#1E1535` | Fondo de todas las cards/paneles |
| **Fondo tarjetas hover** | `#251A3A` | Cards en hover / nivel 2 |
| **Borde tarjetas** | `#2E2050` | Borde `1px solid` en los 4 lados de cada card |
| **Naranja oscuro** | `#C94E00` | Parte baja del gradiente naranja |
| **Púrpura claro** | `#7B35CC` | Degradado púrpura claro, badges |
| **Texto principal** | `#FFFFFF` | Valores KPI, títulos |
| **Texto secundario** | `#8A7FA0` | Labels uppercase, subtítulos, fechas |
| **Texto etiquetas** | `#B0A8C0` | Ejes de gráficos, etiquetas de leyenda |
| **Verde positivo** | `#00C896` | Variaciones positivas ↑ |
| **Rojo negativo** | `#E84545` | Variaciones negativas ↓ |

### Tipografía (NO NEGOCIABLE)

Según el Manual de Identidad XSell:

- **Fuente Principal — `Sora`** (Google Fonts): moderna, geométrica, limpia. Para títulos, subtítulos, KPIs, headers. Pesos: Ultra Delgada → Extranegrita.
- **Fuente Secundaria — `Manrope`** (Google Fonts): moderna, premium, tech. Para cuerpo, descripciones, etiquetas, datos. Pesos: Muy Delgada → Extranegrita.
- **Fuente de Impacto — `Anton`** (Google Fonts): para frases clave, campañas, elementos de alto impacto visual. Solo variable Regular.

- Import obligatorio en todo HTML:
```html
<link href="https://fonts.googleapis.com/css2?family=Anton&family=Sora:wght@300;400;600;700;800&family=Manrope:wght@300;400;500;600;700;800&display=swap" rel="stylesheet">
```

### Logo XSell (imagen base64 oficial — NO usar SVG)

**REGLA CRÍTICA**: NUNCA construir el logo XSell como SVG. El logo tiene gradientes y formas complejas que no pueden replicarse fielmente. Siempre usar la imagen base64 oficial extraída del Manual de Identidad.

**Logo horizontal** (uso principal — headers de dashboards, fondo oscuro o claro):
```html
<img src="data:image/png;base64,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CIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgCIIgiP9M/h96brD/UkpE6gAAAABJRU5ErkJggg==" alt="XSell" height="40" style="display:block">
```

**Reglas de uso según el Manual de Identidad XSell:**
- Logo horizontal: usar en headers de dashboards (fondo oscuro `#1A1025` o blanco)
- PNG con fondo transparente: funciona sobre ambos fondos
- No existe versión reducida — siempre usar la imagen completa
- Área de respeto: dejar espacio equivalente al isotipo alrededor del logo
### Reglas de Aplicación de Color

- El **fondo página** es `#1A1025`. El **fondo de tarjetas** es `#1E1535` (ligeramente más claro). Sin excepciones.
- El **naranja `#FF6B13`** es el color dominante: bordes superiores de resumen ejecutivo, variaciones positivas, barras principales, líneas de gráfico, CTAs.
- El **púrpura `#5D17A0` / `#7B35CC`** es el color secundario: gradientes de área, elementos de embudo, tracks de gauge, badges de íconos.
- **Gradiente naranja→morado**: aplicar en todas las áreas de gráfico de tendencia temporal. Naranja arriba, morado en el medio, casi transparente abajo.
- Variaciones positivas: `#00C896` (verde). Negativas: `#E84545` (rojo).
- Bordes de tarjetas: `1px solid #2E2050` — sutiles, uniformes en los 4 lados. **Sin borde izquierdo de color**.
- Grid lines de gráficos: `rgba(255,255,255,0.05)` — casi invisibles.
- Nunca usar fondo blanco, gris claro, ni colores fuera de la paleta XSell.

### Header Obligatorio

```html
<!-- Header XSell — patrón exacto -->
<header style="display:flex; align-items:center; justify-content:space-between;
               padding:16px 24px; border-bottom:1px solid #2E2050; margin-bottom:24px;">
  <div style="display:flex; align-items:center; gap:16px;">
    <img src="data:image/png;base64,{LOGO_B64}" alt="XSell" height="40" style="display:block">
    <div>
      <div style="font-family:Sora,sans-serif; font-size:11px; font-weight:600;
                  color:#8A7FA0; text-transform:uppercase; letter-spacing:1px;">
        Planeamiento y Control
      </div>
      <div style="font-family:Sora,sans-serif; font-size:17px; font-weight:700; color:#fff;">
        {TÍTULO DEL DASHBOARD}
      </div>
    </div>
  </div>
  <span style="font-size:12px; color:#8A7FA0;">{FECHA}</span>
</header>
```

- El logo es la imagen base64 oficial (ver sección Logo XSell). Nunca construir SVG.
- "Planeamiento y Control" en `#8A7FA0` uppercase pequeño encima del título.
- La fecha de generación va a la derecha en `#8A7FA0`.

---

## Arquitectura del Dashboard

> **REFERENCIA VISUAL OFICIAL**: El dashboard del Manual de Identidad XSell (página 18) es la referencia de oro para el diseño. Replicar fielmente su estética: fondo muy oscuro, tarjetas con bordes sutiles, gradientes naranja→morado en gráficos, tipografía Sora/Manrope, valores KPI grandes en blanco.

### CSS Base Obligatorio (copiar en todo dashboard HTML)

```css
@import url('https://fonts.googleapis.com/css2?family=Anton&family=Sora:wght@300;400;600;700;800&family=Manrope:wght@300;400;500;600;700;800&display=swap');

:root {
  --bg:         #1A1025;   /* fondo página */
  --bg-card:    #1E1535;   /* fondo tarjetas */
  --bg-card2:   #251A3A;   /* fondo tarjetas hover / nivel 2 */
  --border:     #2E2050;   /* borde sutil entre tarjetas */
  --orange:     #FF6B13;   /* naranja primario */
  --orange-dark:#C94E00;   /* naranja oscuro (gradiente) */
  --purple:     #5D17A0;   /* púrpura primario */
  --purple-light:#7B35CC;  /* púrpura claro (gradiente) */
  --text:       #FFFFFF;
  --text-muted: #8A7FA0;   /* etiquetas, subtítulos */
  --text-label: #B0A8C0;   /* labels de gráficos */
  --green:      #00C896;   /* variación positiva */
  --red:        #E84545;   /* variación negativa */
}

* { box-sizing: border-box; margin: 0; padding: 0; }
body {
  background: var(--bg);
  color: var(--text);
  font-family: 'Manrope', sans-serif;
  font-size: 14px;
  min-height: 100vh;
  padding: 20px;
}

/* Tarjeta base */
.card {
  background: var(--bg-card);
  border: 1px solid var(--border);
  border-radius: 14px;
  padding: 18px 20px;
  position: relative;
  overflow: hidden;
}

/* Badge de ícono (esquina superior derecha de KPI cards) */
.kpi-icon-badge {
  width: 36px; height: 36px;
  border-radius: 10px;
  display: flex; align-items: center; justify-content: center;
  font-size: 16px;
  position: absolute; top: 16px; right: 16px;
}
.kpi-icon-badge.orange { background: rgba(255,107,19,0.25); color: var(--orange); }
.kpi-icon-badge.purple { background: rgba(93,23,160,0.35);  color: var(--purple-light); }

/* Valor KPI grande */
.kpi-value {
  font-family: 'Sora', sans-serif;
  font-size: 28px; font-weight: 700;
  color: var(--text);
  margin: 8px 0 4px;
  line-height: 1.1;
}

/* Variación */
.kpi-delta { font-size: 13px; font-weight: 600; }
.kpi-delta.up   { color: var(--green); }
.kpi-delta.down { color: var(--red); }

/* Etiqueta superior de KPI */
.kpi-label {
  font-size: 11px; font-weight: 600;
  color: var(--text-muted);
  text-transform: uppercase; letter-spacing: 0.8px;
}

/* ---------- Responsive (OBLIGATORIO en todo dashboard) ---------- */
/* Grid fluido: usa .grid para los contenedores de tarjetas/gráficos.
   Colapsa 4 → 2 (tablet) → 1 (móvil). Para cards que abarcan 2 columnas usa .col-2. */
.grid { display: grid; gap: 16px; grid-template-columns: repeat(4, 1fr); }
.col-2 { grid-column: span 2; }
.col-4 { grid-column: span 4; }

/* Contenedor de gráfico con alto fijo → Chart.js escala bien con maintainAspectRatio:false */
.chart-box { position: relative; width: 100%; height: 300px; }

/* Tablet: 4 → 2 columnas */
@media (max-width: 1024px) {
  .grid { grid-template-columns: repeat(2, 1fr); }
  .col-2, .col-4 { grid-column: span 2; }
}

/* Móvil: todo a 1 columna, tipografía y paddings reducidos */
@media (max-width: 640px) {
  body { padding: 12px; font-size: 13px; }
  .grid { grid-template-columns: 1fr; gap: 12px; }
  .col-2, .col-4 { grid-column: span 1; }
  .card { padding: 14px 16px; border-radius: 12px; }
  .kpi-value { font-size: 22px; }
  .kpi-label { font-size: 10px; }
  .kpi-icon-badge { width: 30px; height: 30px; top: 12px; right: 12px; font-size: 14px; }
  .chart-box { height: 240px; }
  /* Header apilado: logo/título arriba, fecha abajo */
  header { flex-direction: column; align-items: flex-start !important; gap: 8px; }
  /* Tablas anchas: scroll horizontal propio, no romper el layout */
  .table-scroll { overflow-x: auto; -webkit-overflow-scrolling: touch; }
  table { min-width: 520px; }
}
```

### Optimización Móvil (OBLIGATORIO)

Los dashboards se abren tanto en desktop como desde el celular. Todo dashboard debe verse y funcionar bien en móvil, sin scroll horizontal accidental ni texto diminuto. Reglas:

1. **Meta viewport.** Incluir SIEMPRE en el `<head>`: `<meta name="viewport" content="width=device-width, initial-scale=1.0">`. Sin él el móvil renderiza a ~980px y todo se ve minúsculo. Es el error #1.
2. **Grid fluido, nunca anchos fijos en px.** Los contenedores de KPIs y gráficos usan `.grid` (ya en el CSS base): colapsa 4 → 2 columnas (≤1024px) → 1 columna (≤640px). Para cards que abarcan 2 columnas usa `.col-2`. Nunca fijes `width:900px` u offsets absolutos que rompan en pantallas angostas.
3. **Gráficos en `.chart-box`.** Cada `<canvas>` va dentro de `<div class="chart-box">` y Chart.js corre con `maintainAspectRatio:false`. Así el gráfico llena el ancho y el CSS reduce su alto en móvil (300px → 240px). Sin `width`/`height` fijos en el `<canvas>`.
4. **Tablas con scroll propio.** Ranking y detalle no caben en 1 columna: envolver en `<div class="table-scroll"><table>…</table></div>` para que solo la tabla haga scroll horizontal.
5. **Toques cómodos.** Botones, filtros y selects con altura mínima ~40px y `font-size` ≥14px (evita el zoom automático de iOS al enfocar inputs). No dependas del hover para mostrar datos clave: en móvil no hay hover.
6. **Header apilado.** En móvil el header pasa a columna (logo+título arriba, fecha abajo) — ya cubierto por el CSS base. Logo a `height:40px`, no agrandarlo en móvil.
7. **Verificación.** Antes de entregar, revisar mentalmente a 375px de ancho (iPhone estándar): ¿hay scroll horizontal? ¿algún KPI se corta? ¿los gráficos se leen? Si algo se desborda, es un ancho fijo olvidado.

### Gradientes para Gráficos Chart.js (patrón oficial)

Los gráficos de área/línea usan gradiente vertical **naranja arriba → morado abajo → transparente**:

```javascript
function makeGradient(ctx, chartArea) {
  if (!chartArea) return 'rgba(255,107,19,0.4)';  // primer render: chartArea aún no existe
  const grad = ctx.createLinearGradient(0, chartArea.top, 0, chartArea.bottom);
  grad.addColorStop(0,   'rgba(255,107,19, 0.85)');   /* naranja sólido arriba */
  grad.addColorStop(0.4, 'rgba(93,23,160,  0.60)');   /* morado al medio */
  grad.addColorStop(1,   'rgba(93,23,160,  0.05)');   /* casi transparente abajo */
  return grad;
}
```

⚠️ **NUNCA llamar `makeGradient(ctx, chart.chartArea)` directamente en el dataset** — en el primer render `chart.chartArea` es `undefined` y el gradiente sale en naranja plano. Usar la forma **scriptable** de Chart.js, que se re-evalúa en cada layout:

```javascript
backgroundColor: function(context) {
  const chart = context.chart;
  const {ctx, chartArea} = chart;
  return makeGradient(ctx, chartArea);   // se recalcula cuando chartArea ya existe
},
borderColor: '#FF6B13', borderWidth: 2, fill: true, tension: 0.4
```

Para **barras**: naranja `#FF6B13` (barras principales) / morado `#5D17A0` (secundarias).
Para **donuts/pie**: primero naranja, segundo morado, tercero `#7B35CC`.
Para **embudo**: de arriba abajo — morado `#5D17A0` → intermedio → naranja `#FF6B13`.
Para **gauge circular**: track morado `#2E2050`, progreso naranja→morado.

### Estructura Estándar (4 secciones)

Cada dashboard sigue esta estructura de arriba hacia abajo:

**SECCIÓN 1 — KPIs Resumen (parte superior)**
- 4-6 tarjetas `.card` en grid horizontal (igual ancho)
- Cada tarjeta tiene: etiqueta uppercase pequeña (`.kpi-label`) + badge ícono (`.kpi-icon-badge`) en esquina superior derecha + valor grande (`.kpi-value`) + variación con color (`.kpi-delta`) + texto "vs mes anterior" gris + mini sparkline Chart.js en la parte inferior de la tarjeta (altura 50px, sin ejes, sin grid, sin tooltip excepto hover)
- **SIN borde de color a la izquierda** — las cards son uniformes con borde `1px solid var(--border)` en todos los lados
- El sparkline usa el gradiente naranja→morado

**SECCIÓN 2 — Gráficos Analíticos (cuerpo principal)**
- Grid de 2-3 columnas, cards de mayor altura (250-300px de área de gráfico)
- Card grande (span 2 cols): gráfico de área con gradiente naranja→morado — tendencia temporal principal
- Cards medianas: embudo de conversión (morado→naranja), gauge de meta circular, donut de distribución
- Cards pequeñas: tabla de ranking con posición numerada, lista de actividad con métricas + delta
- Todos los gráficos sobre fondo `var(--bg-card)`, grid lines `rgba(255,255,255,0.05)`, sin bordes de ejes
- Colores: naranja `#FF6B13` protagonista, púrpura `#5D17A0` secundario

**SECCIÓN 3 — Predicción de Escenarios**
- Proyección a 3-6 meses, gráfico de líneas con 3 escenarios
- Línea base: naranja `#FF6B13` / Optimista: verde `#00C896` / Pesimista: rojo `#E84545`
- Zona de confianza: `rgba(255,107,19,0.08)` entre optimista y pesimista
- Metodología explicada en texto pequeño debajo del gráfico

**SECCIÓN 4 — Resumen Ejecutivo: Hallazgos, Estrategias y Conclusiones**

Panel `.card` con fondo `var(--bg-card)`, borde superior `3px solid var(--orange)`, tres bloques separados por `1px dashed var(--border)`.

**Bloque 4.1 — Hallazgos Clave** (ícono 🔍)
- 3-5 hallazgos derivados directamente de los datos analizados
- Cada hallazgo debe ser un hecho concreto y cuantificado, NO una opinión
- Formato: número circular naranja + hallazgo en negrita + dato de respaldo
- Priorizar por impacto: del hallazgo más crítico al menos urgente

**Bloque 4.2 — Estrategias Recomendadas** (ícono 🎯)
- 3-5 estrategias que responden directamente a los hallazgos
- Cada estrategia: acción específica + responsable sugerido + plazo + KPI objetivo
- Las estrategias deben ser ejecutables, no genéricas

**Bloque 4.3 — Conclusiones** (ícono 📋)
- 2-3 conclusiones de tono gerencial
- Responder: ¿estamos bien, mal o en riesgo? ¿Tendencia? ¿Qué pasa si no se actúa?

**Diseño visual:** Fondo `var(--bg-card)`, borde superior `3px solid var(--orange)`, separadores `1px dashed var(--border)`.

---

## Implementación Técnica

### Para dashboards HTML (archivo único autocontenido)

Usa Chart.js. El CSS base está en la sección "Arquitectura del Dashboard" — siempre incluirlo completo.

```html
<!-- OBLIGATORIO en el <head>: sin este meta el dashboard se ve diminuto en móvil -->
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<!-- CDN obligatorios -->
<link href="https://fonts.googleapis.com/css2?family=Anton&family=Sora:wght@300;400;600;700;800&family=Manrope:wght@400;500;600;700&display=swap" rel="stylesheet">
<script src="https://cdnjs.cloudflare.com/ajax/libs/Chart.js/4.4.1/chart.umd.js"></script>
```

**Configuración global Chart.js:**
```javascript
Chart.defaults.font.family = "'Manrope', sans-serif";
Chart.defaults.color = '#B0A8C0';
Chart.defaults.responsive = true;
Chart.defaults.maintainAspectRatio = false;   // el gráfico llena su .chart-box (alto por CSS); clave para móvil
Chart.defaults.plugins.legend.labels.usePointStyle = true;
Chart.defaults.plugins.legend.labels.boxWidth = 8;
Chart.defaults.scale.grid.color = 'rgba(255,255,255,0.05)';
Chart.defaults.scale.border.display = false;
Chart.defaults.scale.ticks.color = '#8A7FA0';
```

> Envuelve cada `<canvas>` en un `<div class="chart-box">`. Con `maintainAspectRatio:false` el gráfico se adapta a ese contenedor y el CSS responsive reduce su alto en móvil (300px → 240px). No pongas `width`/`height` fijos como atributos del `<canvas>`.

**Paleta para datasets:**
```javascript
const PALETTE_XSELL = {
  orange:        '#FF6B13',
  orangeDark:    '#C94E00',
  purple:        '#5D17A0',
  purpleLight:   '#7B35CC',
  green:         '#00C896',
  red:           '#E84545',
  yellow:        '#F6A500',
  text:          '#FFFFFF',
  textMuted:     '#8A7FA0',
  textLabel:     '#B0A8C0',
  border:        '#2E2050',
  bg:            '#1A1025',
  bgCard:        '#1E1535',
  /* Gradiente estándar — crear con makeGradient() */
};

function makeGradient(ctx, chartArea) {
  if (!chartArea) return '#FF6B13';
  const g = ctx.createLinearGradient(0, chartArea.top, 0, chartArea.bottom);
  g.addColorStop(0,   'rgba(255,107,19, 0.85)');
  g.addColorStop(0.4, 'rgba(93,23,160,  0.60)');
  g.addColorStop(1,   'rgba(93,23,160,  0.05)');
  return g;
}
```

### Para dashboards en Excel (.xlsx)

Consulta el skill `xlsx` para reglas de formato de Excel. Además:
- Pestaña "Dashboard" con gráficos y KPIs
- Pestaña "Data" con datos fuente
- Pestaña "Predicciones" con los cálculos de escenarios
- Pestaña "Conclusiones" con las estrategias
- Aplicar colores corporativos a todos los gráficos y celdas de encabezado

---

## Motor Analítico y Predictivo

### Procesamiento de Datos
1. **Inspeccionar**: Primeras filas, tipos, nulos, outliers
2. **Limpiar**: Valores faltantes, normalizar fechas, deduplicar
3. **Transformar**: Métricas derivadas (tasas, promedios móviles, acumulados)
4. **Analizar**: Patrones, correlaciones, estacionalidades, anomalías

### Predicción de Escenarios

Regresión lineal + desviación estándar para 3 escenarios (optimista = base + 1.5σ, base, pesimista = base − 1.5σ). Para datos categóricos, analizar tendencia de composición porcentual.

⚠️ **El dashboard HTML corre en el navegador — NO hay numpy/pandas/Python.** La proyección debe escribirse en **JavaScript puro**. Patrón de mínimos cuadrados sin librerías:

```javascript
// Regresión lineal simple sobre serie temporal (x = índice de mes, y = valor)
function linReg(values) {
  const n = values.length;
  const xs = values.map((_, i) => i);
  const sumX = xs.reduce((a, b) => a + b, 0);
  const sumY = values.reduce((a, b) => a + b, 0);
  const sumXY = xs.reduce((a, x, i) => a + x * values[i], 0);
  const sumX2 = xs.reduce((a, x) => a + x * x, 0);
  const slope = (n * sumXY - sumX * sumY) / (n * sumX2 - sumX * sumX);
  const intercept = (sumY - slope * sumX) / n;
  // desviación estándar de los residuos
  const resid = values.map((y, i) => y - (slope * i + intercept));
  const sigma = Math.sqrt(resid.reduce((a, r) => a + r * r, 0) / n);
  return { slope, intercept, sigma };
}
// Proyección mes futuro k: base = slope*(n+k) + intercept; optimista = base + 1.5*sigma; pesimista = base - 1.5*sigma
```

Si el análisis se hace **offline en el contenedor** (no en el dashboard) para generar insights del resumen ejecutivo, ahí sí se puede usar numpy/pandas con Python. La distinción: **HTML entregable = JS puro; análisis previo de Claude = Python permitido.**

### Resumen Ejecutivo (framework encadenado)
**Hallazgos** (dato→hecho): cuantificados, comparados con benchmark, priorizados por impacto
**Estrategias** (hecho→acción): cada una con acción concreta + plazo + KPI objetivo + hallazgo vinculado
**Conclusiones** (análisis→veredicto): estado general, tendencia, impacto si no se actúa

---

## SQL — Normalización, Scripts y Queries

### Reglas
- Normalizar a 3FN mínimo. Generar diagrama ER + DDL + documentación
- Estilo: palabras clave MAYÚSCULAS, tablas snake_case plural, columnas snake_case, alias significativos, indentación 4 espacios
- Comentarios: explicar el POR QUÉ, no el QUÉ

### Queries frecuentes BPO
- Volumen por canal/periodo, SLA, productividad por agente, abandono, CSAT/NPS/FCR, forecast de demanda

---

## Integración con Supabase

Proyecto: `ejxpojzuovtsplviafbw`.

### Reglas Clave
- NUNCA usar `apply_migration` — siempre `execute_sql` para DDL (`apply_migration` lanza `UnauthorizedException`)
- Fechas: siempre declarar como `TEXT` en el DDL (MySQL puede tener `0000-00-00`)
- INSERTs máximo 500 filas por lote
- Crear índices en columnas de filtro frecuente (fechas, estados, IDs)

### Flujo MySQL → Supabase (el más común)
1. `dashboard-query?action=search&q=palabra` → encontrar tabla en MySQL
2. `dashboard-query?action=describe&database=X&table=Y` → ver columnas
3. `Supabase:execute_sql` con el DDL → crear tabla (fechas como TEXT, NUNCA `apply_migration`)
4. Disparar sync con `pg_net.http_get` dentro de `execute_sql` → `dashboard-query?action=sync_execute...`
5. Verificar: `Supabase:execute_sql` → `SELECT COUNT(*) FROM tabla`
6. Preguntar frecuencia de auto-sync al usuario

### Flujo Excel/CSV → Supabase
1. Leer archivo con Python → analizar estructura
2. Diseñar esquema normalizado (1FN→3FN)
3. `Supabase:execute_sql` con DDL → luego INSERTs por lotes ≤500

---

## Conexión Dashboard → Supabase: Carga de Datos (CRÍTICO)

### El problema más común: dashboard en blanco

Los dashboards quedan en blanco cuando se intenta conectar a Supabase de forma incorrecta. **Causa raíz**: el navegador bloquea cualquier request con headers `apikey` o `Authorization` desde un archivo HTML local (`file://`) o desde Netlify sin configuración especial de CORS. **Solución única y obligatoria**: usar siempre la Edge Function `dashboard-data` como proxy — nunca la API REST de Supabase directamente.

### Regla de oro — `dashboard-data` v2 es el único punto de acceso

```
const EF = 'https://ejxpojzuovtsplviafbw.supabase.co/functions/v1/dashboard-data';
```

Esta URL es la **única constante** que va en los dashboards. Sin API keys, sin tokens, sin headers especiales. La EF usa `service_role` internamente en el servidor — los analistas nunca la ven.

**NUNCA usar directamente:**
- `https://ejxpojzuovtsplviafbw.supabase.co/rest/v1/...` → requiere `apikey` → CORS bloqueado
- `supabase-js` client en el browser → mismo problema
- Cualquier header `Authorization` o `apikey` en fetch desde HTML

---

### API Completa de `dashboard-data` v2

#### `?action=tables` — Listar todas las tablas disponibles
```javascript
const { tables } = await fetch(EF + '?action=tables').then(r => r.json());
// tables[i]: { supabase_table, mysql_database, mysql_table, total_filas, ultimo_sync, es_sistema }
```

#### `?action=columns&table=X` — Columnas de una tabla
```javascript
const { columns } = await fetch(EF + '?action=columns&table=gestiones_scaniaarg').then(r => r.json());
// columns: ['tipo_resultado', 'nombre_usuario', 'duracion', ...]
```

#### `?action=values&table=X&column=Y` — Valores únicos de una columna (para poblar filtros)
```javascript
const { values } = await fetch(EF + '?action=values&table=gestiones_scaniaarg&column=tipo_resultado').then(r => r.json());
// values: ['VENTA', 'NO VENTA', 'NO CONTESTA', ...]  — ya ordenados
// Parámetro opcional: &limit=200 (default 500, máx 1000)
```

#### `?action=download&table=X` — Descarga toda la tabla (sin filtros)
```javascript
const json = await fetch(EF + '?action=download&table=gestiones_scaniaarg').then(r => r.json());
const rows = (json.d && json.d.data) || [];
// Descarga paginada internamente, devuelve todo. Máx 200k filas.
```

⚠️ **Regla de tamaño:** antes de usar `download`, verificar `total_filas` de la tabla (vía `action=tables` o `count_only=true`). Si supera **50,000 filas**, NO usar `download` — el browser puede hacer timeout o recibir respuesta parcial sin error. En ese caso usar `action=query` con paginación (`limit` + `offset`) o filtros server-side que reduzcan el set. Para dashboards, casi nunca se necesitan todas las filas crudas en el cliente: agregar/contar server-side cuando sea posible.

#### `?action=query` — Query con filtros, columnas, orden y paginación ⭐
El más importante para dashboards interactivos. Soporta GET y POST.

**GET — parámetros por query string:**
```
?action=query
&table=gestiones_scaniaarg
&columns=tipo_resultado,nombre_usuario,duracion,fecha    ← columnas (omitir = *)
&limit=500                                               ← máx 50000, default 1000
&offset=0                                               ← paginación
&order_by=duracion&order_dir=desc                       ← orden
&filter_col=tipo_resultado&filter_val=VENTA             ← filtro exacto 1
&filter_col2=nombre_usuario&filter_val2=MARIA           ← filtro exacto 2
&date_col=fecha&date_from=2026-01-01&date_to=2026-03-31 ← rango de fechas
&search_col=nombre_usuario&search_val=garcia            ← búsqueda parcial (ilike)
&count_only=true                                        ← solo contar, sin datos
```

**POST — body JSON (más potente, filtros complejos):**
```javascript
const json = await fetch(EF + '?action=query', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    table: 'gestiones_scaniaarg',
    columns: 'tipo_resultado,nombre_usuario,duracion,fecha',
    limit: 1000,
    offset: 0,
    order_by: 'fecha',
    order_dir: 'desc',
    filters: [
      { col: 'tipo_resultado', op: 'eq',    val: 'VENTA' },
      { col: 'duracion',       op: 'gte',   val: 30 },
      { col: 'nombre_usuario', op: 'ilike', val: '%garcia%' },
      { col: 'tipo_resultado', op: 'in',    val: ['VENTA', 'INTERESADO'] }
    ]
  })
}).then(r => r.json());

const rows  = (json.d && json.d.data) || [];
const total = json.total;   // total de filas que cumple los filtros (para paginación)
const limit = json.limit;
const offset = json.offset;
```

**Operadores disponibles en filters:** `eq`, `neq`, `gt`, `gte`, `lt`, `lte`, `like`, `ilike`, `in`

**Respuesta de `query` y `download`:**
```json
{ "d": { "data": [...] }, "total": 1532, "limit": 1000, "offset": 0 }
```
Acceder siempre con `(json.d && json.d.data) || []`

---

### ⚠ FORMATO DE RESPUESTA — campo exacto por acción

| Acción | Campo de datos | Ejemplo acceso |
|--------|----------------|----------------|
| `tables` | `json.tables` | `const tables = json.tables \|\| []` |
| `columns` | `json.columns` | `const cols = json.columns \|\| []` |
| `values` | `json.values` | `const vals = json.values \|\| []` |
| `download` | `json.d.data` | `const rows = (json.d && json.d.data) \|\| []` |
| `query` | `json.d.data` | `const rows = (json.d && json.d.data) \|\| []` |

**El error más común:** usar `json.data` en vez de `json.d.data` para download/query → dashboard en blanco sin error.

### Patrón correcto completo para cargar datos del dashboard

```javascript
const EF = 'https://ejxpojzuovtsplviafbw.supabase.co/functions/v1/dashboard-data';

// ── CARGA INICIAL: tabla completa (tablas pequeñas/medianas) ──
async function loadDashboard() {
  try {
    showLoading(true);
    const json = await fetch(EF + '?action=download&table=MI_TABLA').then(r => r.json());
    ALL_ROWS = (json.d && json.d.data) || [];
    if (!ALL_ROWS.length) { showError('Sin datos. Verifica que la tabla esté sincronizada.'); return; }
    await populateFilters();   // poblar selects con valores reales de la BD
    applyFilters();
  } catch(e) { showError('Error: ' + e.message); }
  finally { showLoading(false); }
}

// ── CARGA CON FILTROS SERVER-SIDE (tablas grandes, paginación) ──
async function loadWithFilters(filterState) {
  try {
    showLoading(true);
    const json = await fetch(EF + '?action=query', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        table: 'MI_TABLA',
        columns: 'tipo_resultado,nombre_usuario,duracion,fecha',
        limit: 1000,
        offset: filterState.offset || 0,
        order_by: 'fecha', order_dir: 'desc',
        filters: filterState.filters || []
      })
    }).then(r => r.json());
    const rows  = (json.d && json.d.data) || [];
    const total = json.total;
    renderKPIs(rows);
    renderCharts(rows);
    updateContador(rows.length, total);
  } catch(e) { showError('Error: ' + e.message); }
  finally { showLoading(false); }
}

// ── POBLAR FILTROS DESDE LA BD (no desde los datos locales) ──
async function populateFilters() {
  const [r1, r2] = await Promise.all([
    fetch(EF + '?action=values&table=MI_TABLA&column=tipo_resultado').then(r => r.json()),
    fetch(EF + '?action=values&table=MI_TABLA&column=nombre_usuario').then(r => r.json())
  ]);
  const selRes = document.getElementById('filtroResultado');
  selRes.innerHTML = '<option value="">Todos</option>' +
    (r1.values || []).map(v => `<option value="${v}">${v}</option>`).join('');
  const selAg = document.getElementById('filtroAgente');
  selAg.innerHTML = '<option value="">Todos</option>' +
    (r2.values || []).map(v => `<option value="${v}">${v}</option>`).join('');
}
```

Las columnas de Supabase son **siempre minúsculas**: `r.tipo_resultado` ✅ — `r.TIPO_RESULTADO` ❌

### Cargar múltiples tablas en paralelo

```javascript
async function loadDashboard() {
  try {
    // fetch en paralelo — mucho más rápido que secuencial
    const [r1, r2] = await Promise.all([
      fetch(EF + '?action=download&table=gestiones_scaniaarg'),
      fetch(EF + '?action=download&table=asistencias_dashboard')
    ]);
    const [d1, d2] = await Promise.all([r1.json(), r2.json()]);

    const gestiones   = (d1.d && d1.d.data) || [];
    const asistencias = (d2.d && d2.d.data) || [];

    renderKPIs(gestiones, asistencias);
    renderCharts(gestiones);
  } catch (err) {
    showError('Error: ' + err.message);
  }
}
```

### Skeleton / loading obligatorio

El dashboard NUNCA debe aparecer en blanco mientras carga. Siempre mostrar estado de carga:

```javascript
// showLoading SOLO pinta el skeleton. NO intenta restaurar valores —
// de eso se encarga renderKPIs() que escribe el valor real directamente.
function showLoading(active) {
  if (!active) return;  // al desactivar, renderKPIs ya sobrescribe el innerHTML
  document.querySelectorAll('.kpi-value').forEach(el => {
    el.innerHTML = '<span style="display:inline-block;width:80px;height:28px;background:var(--border);border-radius:6px;animation:pulse 1.2s infinite"></span>';
  });
}

// renderKPIs escribe SIEMPRE el valor calculado — esto borra el skeleton.
// Ejemplo: document.getElementById('kpi-ventas').textContent = totalVentas;

function showError(msg) {
  document.getElementById('dashboard-error').style.display = 'block';
  document.getElementById('dashboard-error').textContent = '⚠ ' + msg;
}
```

⚠️ El skeleton se quita porque `renderKPIs()` reescribe el `innerHTML`/`textContent` de cada KPI con su valor real. Si un KPI nunca se actualiza en `renderKPIs()`, se queda con el skeleton para siempre — asegurar que cada `.kpi-value` del HTML tenga su línea correspondiente en `renderKPIs()`.

```css
@keyframes pulse {
  0%, 100% { opacity: 0.4; }
  50%       { opacity: 1;   }
}
```

### Errores frecuentes y cómo diagnosticarlos

| Síntoma | Causa | Solución |
|---------|-------|----------|
| Dashboard en blanco, sin error visible | `fetch` sin `.catch()` — el error se silencia | Agregar `.catch(err => showError(err.message))` siempre |
| Dashboard en blanco con datos cargados | Accediendo a `json.data` en vez de `json.d.data` | Para `query`/`download`: usar `(json.d && json.d.data) \|\| []` |
| `TypeError: Cannot read property of undefined` | Columnas en MAYÚSCULAS en JS | Supabase es siempre minúsculas: `r.tipo_resultado` no `r.TIPO_RESULTADO` |
| `NetworkError` / `CORS` en consola | Usando la API REST de Supabase directamente | Cambiar a `dashboard-data` EF — es el único acceso correcto |
| `401 Unauthorized` | Usando endpoint con JWT requerido | `dashboard-data` no requiere JWT (verify_jwt: false) |
| Filtros no tienen efecto | Usando `download` con params extra | Para filtros server-side usar `action=query` con `filters: [...]` |
| Select vacío después de poblar | Usando datos locales para poblar en vez de la BD | Usar `action=values&table=X&column=Y` — devuelve valores únicos directamente de Supabase |
| Datos cargados pero gráfico vacío | Columnas `null` o tipo incorrecto | Filtrar: `rows.filter(r => r.columna != null)` antes de graficar |
| Tabla vacía — sync no corrido | La tabla existe en Supabase pero sin datos | Verificar con `action=query&count_only=true` — si total=0, correr sync |

### Filtros y Búsquedas en el Dashboard

**¿Server-side (`action=query`) o client-side (JS sobre el array)?**

| Situación | Recomendación |
|-----------|---------------|
| Tabla ≤ 20k filas | Client-side — descargar con `download`, filtrar en JS |
| Tabla > 20k filas | Server-side — usar `query` con `filters: [...]` |
| Filtros dependientes (el valor de uno afecta las opciones del otro) | Server-side |
| Dashboard con paginación | Server-side — usar `limit` + `offset` + `total` |
| Búsqueda de texto libre | Server-side — `{ op: 'ilike', val: '%texto%' }` |

**Patrón client-side completo (tablas pequeñas/medianas):**

```javascript
// ── ESTADO GLOBAL ─────────────────────────────────────────────
let ALL_ROWS = [];       // datos originales completos — nunca mutar
let filteredRows = [];   // vista filtrada actual

async function loadDashboard() {
  const json = await fetch(EF + '?action=download&table=MI_TABLA').then(r => r.json());
  ALL_ROWS = (json.d && json.d.data) || [];
  applyFilters();  // renderizar con filtros iniciales (= todos)
}

// ── FUNCIÓN CENTRAL DE FILTRADO ───────────────────────────────
function applyFilters() {
  const mesVal      = document.getElementById('filtroMes').value;
  const agenteVal   = document.getElementById('filtroAgente').value.toLowerCase();
  const resultadoVal = document.getElementById('filtroResultado').value;

  filteredRows = ALL_ROWS.filter(r => {
    // Filtro por mes (columna tipo TEXT "2026-04-15 10:30:00")
    const okMes = !mesVal || (r.fecha && r.fecha.startsWith(mesVal));

    // Filtro por agente (búsqueda parcial, case-insensitive)
    const okAgente = !agenteVal ||
      (r.nombre_usuario && r.nombre_usuario.toLowerCase().includes(agenteVal));

    // Filtro por resultado exacto
    const okResultado = !resultadoVal || r.tipo_resultado === resultadoVal;

    return okMes && okAgente && okResultado;
  });

  // Re-renderizar todo con los datos filtrados
  renderKPIs(filteredRows);
  renderCharts(filteredRows);
  updateContador(filteredRows.length, ALL_ROWS.length);
}

// ── POBLAR SELECTS CON VALORES ÚNICOS ─────────────────────────
function populateFilters(rows) {
  // Meses únicos
  const meses = [...new Set(
    rows.map(r => r.fecha ? r.fecha.substring(0, 7) : null).filter(Boolean)
  )].sort();
  const selMes = document.getElementById('filtroMes');
  selMes.innerHTML = '<option value="">Todos los meses</option>' +
    meses.map(m => `<option value="${m}">${m}</option>`).join('');

  // Resultados únicos
  const resultados = [...new Set(rows.map(r => r.tipo_resultado).filter(Boolean))].sort();
  const selRes = document.getElementById('filtroResultado');
  selRes.innerHTML = '<option value="">Todos los resultados</option>' +
    resultados.map(v => `<option value="${v}">${v}</option>`).join('');
}
```

**HTML de los controles de filtro** (colocar debajo del header):
```html
<div style="display:flex; gap:12px; flex-wrap:wrap; margin-bottom:20px; align-items:center;">
  <select id="filtroMes" onchange="applyFilters()"
    style="background:var(--bg-card);color:var(--text);border:1px solid var(--border);
           border-radius:8px;padding:8px 12px;font-family:Manrope,sans-serif;font-size:13px;">
  </select>
  <select id="filtroResultado" onchange="applyFilters()"
    style="background:var(--bg-card);color:var(--text);border:1px solid var(--border);
           border-radius:8px;padding:8px 12px;font-family:Manrope,sans-serif;font-size:13px;">
  </select>
  <input id="filtroAgente" oninput="applyFilters()" placeholder="🔍 Buscar asesor..."
    style="background:var(--bg-card);color:var(--text);border:1px solid var(--border);
           border-radius:8px;padding:8px 12px;font-family:Manrope,sans-serif;font-size:13px;
           min-width:200px;">
  <span id="contadorFiltro" style="color:var(--text-muted);font-size:12px;"></span>
  <button onclick="resetFilters()"
    style="background:transparent;color:var(--text-muted);border:1px solid var(--border);
           border-radius:8px;padding:8px 12px;cursor:pointer;font-size:12px;">
    ✕ Limpiar filtros
  </button>
</div>
```

```javascript
function resetFilters() {
  document.getElementById('filtroMes').value = '';
  document.getElementById('filtroResultado').value = '';
  document.getElementById('filtroAgente').value = '';
  applyFilters();
}

function updateContador(filtradas, total) {
  document.getElementById('contadorFiltro').textContent =
    filtradas === total
      ? `${total.toLocaleString('es-PE')} registros`
      : `${filtradas.toLocaleString('es-PE')} de ${total.toLocaleString('es-PE')} registros`;
}
```

**Reglas para filtros:**
- Siempre trabajar sobre `ALL_ROWS` para filtrar — nunca sobre `filteredRows` (pierde datos al encadenar)
- Poblar los selects con `populateFilters(ALL_ROWS)` una sola vez al cargar, no al filtrar
- Los filtros de fecha usan `startsWith('YYYY-MM')` porque las fechas en Supabase son TEXT
- Adaptar los nombres de columna (`r.fecha`, `r.nombre_usuario`, etc.) a las columnas reales de la tabla

---

Antes de escribir una sola línea de HTML, siempre verificar que la tabla tiene datos:

```sql
-- Ejecutar en Supabase execute_sql
SELECT COUNT(*) FROM nombre_tabla;
SELECT * FROM nombre_tabla LIMIT 3;
-- Si COUNT = 0, correr sync primero
```

---

## Exploración Post-Dashboard y Menú del Analista

### Principio: El dashboard es el punto de partida, no el destino final

Después de entregar el dashboard completo, Claude DEBE ofrecer proactivamente una exploración inteligente basada en la data que descubrió.

### Bloque 1: Sugerencias Inteligentes (OBLIGATORIO tras entregar dashboard)

Claude analiza las columnas/tablas que observó y sugiere **3-5 análisis adicionales específicos**:

- Ser **específicas a la data real** que vio (no genéricas)
- Referenciar **campos/columnas/tablas concretas**
- Indicar **qué pregunta de negocio responde**
- Clasificar: 📊 Métrica nueva, 🔄 Cruce de datos, 🎯 Segmentación, 📈 Tendencia, ⚡ Alerta automática

**Framework para generar sugerencias:**
Para cada columna que observó pero NO usó en el dashboard principal:
1. ¿Permite una **segmentación** útil? (región, campaña, turno)
2. ¿Hay un **cruce** entre tablas que agregue valor? (gestiones × chats × asistencias)
3. ¿Hay un **patrón temporal** no explorado? (hora del día, día de semana)
4. ¿Se puede crear una **alerta automática** con umbrales?
5. ¿Los datos permiten un **indicador derivado** no obvio? (ratio recontacto, velocidad de agotamiento de base)

### Bloque 2: Menú de Acciones del Analista

```
🛠️ ¿QUÉ MÁS PUEDO HACER CON ESTA DATA?

📊 Análisis → "Agrégame [métrica]" / "Cruzar [tabla1] con [tabla2]" / "Segmentar por [campo]"
📥 Datos → "Descargar Excel" / "Sincronizar otra tabla" / "Buscar tablas con [palabra]"
📤 Compartir → "Manda resumen por correo" / "Genera PDF" / "Prepara PPT"
⚡ Automatización → "Configura alerta si [métrica] baja de [umbral]" / "Programa sync"
🔍 Exploración → "¿Qué otras tablas tiene [BD]?" / "Describe [tabla]" / "Ejecuta query"
```

### Reglas
1. **Siempre sugerir** — nunca entregar dashboard sin sugerencias post-análisis
2. **Ser específico** — referenciar columnas/tablas reales
3. **Máximo 5 sugerencias**, priorizadas por impacto
4. **Ser ejecutable** — el analista dice "hazme la 3" y Claude lo ejecuta
5. **Mantener contexto** — recordar toda la data descubierta para iterar libremente

---

## Sincronización: Preguntar Frecuencia al Usuario

Después de cada `sync_execute` exitoso, Claude DEBE preguntar al usuario qué frecuencia de auto-sync desea. NO asumir frecuencia automáticamente.

**Pregunta obligatoria:**
"La sincronización fue exitosa. ¿Cada cuánto quieres que se actualice automáticamente esta tabla?"

**Opciones:** Cada hora, Cada 6 horas, Diario (6:00 AM), Semanal, No automatizar (solo manual)

**Contexto:** La mayoría de servicios solo necesitan sync diario. Solo ofrecer "cada hora" si el analista lo pide explícitamente o si el servicio es de alta frecuencia (inbound en tiempo real, por ejemplo).

Usar `dashboard-query?action=sync_auto_config&dest=tabla&frecuencia=diario&hora=06:00` según la respuesta del usuario.

---

## Correo Resumen del Dashboard

Cuando el usuario pida compartir/enviar el dashboard, usar `message_compose` con identidad XSell.

**Asunto**: `[XSell P&C] {Título} — {Fecha}`
**Estructura**: Indicadores Clave (4 KPIs con variación) → Hallazgos Principales (3 cuantificados) → Estrategias (con plazo y KPI) → Conclusión gerencial
**Reglas**: Tono ejecutivo, máximo 250 palabras, datos del dashboard real, ofrecer 2 variantes (informativo / con urgencia)
**Microsoft 365**: Buscar hilos previos con `outlook_email_search`, reportes previos con `sharepoint_search`

---

## Flujo de Trabajo Completo

Cuando el usuario te pida un dashboard, sigue este orden:

1. **Entender el requerimiento**: ¿Qué datos tiene? ¿Qué quiere ver? ¿Para quién es? ¿Qué tipo de servicio? (Inbound/Outbound/Encuestas/Perfilamiento)
2. **Descubrir datos (OBLIGATORIO)**:
   - Preguntar palabras clave al analista
   - Buscar en TODAS las BDs con `dashboard-query?action=search&q=palabra`
   - Identificar tablas: gestión/tipificación primero, luego tiempos, luego asistencia
   - NUNCA asumir BD ni estructura de tablas
3. **Describe y explorar**: `dashboard-query?action=describe` para ver columnas, tipos, sample, valores distintos
4. **Selector de columnas (OBLIGATORIO antes de sincronizar)**: Después del describe, mostrar artefacto interactivo clasificando TODAS las columnas en: ✅ Analíticas (recomendadas), ⬜ Opcionales, 🔒 Datos Personales (solo ranking/filtro interno, no exportar), ⚙️ Operativas. Esperar confirmación del analista — NUNCA sincronizar sin esta confirmación.
5. **Sincronizar a Supabase**: Verificar si tabla existe → aplicar filtro de período (año en curso por defecto) → `execute_sql` DDL → disparar sync vía `pg_net` → polling máx 10 intentos → validar COUNT + rango de fechas → **preguntar frecuencia de auto-sync al usuario**
6. **Procesar**: Limpia, transforma, calcula métricas derivadas
7. **Analizar**: Identifica patrones, tendencias, anomalías
8. **Predecir**: Genera escenarios proyectados (regresión lineal, promedio móvil)
9. **Concluir**: Formula hallazgos, estrategias y conclusiones (Resumen Ejecutivo)
10. **Construir el dashboard**: HTML con identidad XSell + Chart.js + Edge Function `dashboard-data`
11. **Presentar**: Entregar el archivo `.html` al usuario con resumen ejecutivo. El analista lo sube directamente al portal XSell.
12. **Explorar y Sugerir (OBLIGATORIO)**: Sugerencias de análisis adicional específicas + menú de acciones del analista
13. **(Opcional) Enviar por correo**: Si el usuario pide compartir, redactar email con `message_compose`

Si necesitas generar el dashboard como HTML interactivo con Chart.js, crea un archivo `.html` autocontenido. El analista lo sube directamente al portal XSell — no se hace deploy a Netlify. Si necesitas PDF, consulta el skill `pdf`.

---

## Checklist Final (antes de entregar)

- [ ] ¿Se entregó el VisorSQL primero (`https://visor-sql-3eriza.netlify.app`) y se esperó el comando sync?
- [ ] ¿Se siguió el flujo obligatorio de descubrimiento? (search → describe → sync, NUNCA asumir BD)
- [ ] ¿El fondo es `#1A1025` (oscuro XSell)?
- [ ] ¿El naranja `#FF6B13` es el color dominante?
- [ ] ¿Se usan las tipografías Anton / Sora / Manrope?
- [ ] ¿Aparece el logo XSell como imagen base64 oficial (NO SVG construido) y "Planeamiento y Control"?
- [ ] ¿Hay KPIs resumen en la parte superior?
- [ ] ¿Los gráficos muestran tendencias y composición?
- [ ] ¿Hay sección de predicción con escenarios?
- [ ] ¿El Resumen Ejecutivo tiene los 3 bloques: Hallazgos Clave, Estrategias Recomendadas y Conclusiones?
- [ ] ¿Cada hallazgo está cuantificado con datos reales del análisis?
- [ ] ¿Cada estrategia tiene acción, plazo y KPI objetivo vinculado a un hallazgo?
- [ ] ¿Las conclusiones son de tono gerencial con tendencia e impacto si no se actúa?
- [ ] ¿Los colores son exclusivamente de la paleta corporativa?
- [ ] ¿Se preguntó al usuario la frecuencia de auto-sync? (NO asumir automáticamente)
- [ ] ¿Se presentaron sugerencias de análisis adicional específicas a la data descubierta?
- [ ] ¿Se presentó el menú de acciones del analista?
- [ ] ¿Las sugerencias referencian columnas/tablas/campos reales (no genéricas)?
- [ ] ¿Si aplica SPD/HPD: se usó la fórmula correcta (hits ÷ horas logueado/turno) y meta configurable por campaña?
- [ ] ¿Si hay tabla de tiempos: se descubrieron y mapearon estados del CRM origen antes de crear tabla?
- [ ] ¿El SQL (si aplica) está normalizado y bien documentado?
- [ ] ¿Se aplicó filtro de período al sync? (año en curso por defecto — nunca data histórica sin límite)
- [ ] ¿Si se usó Supabase: se verificó que la tabla no existía antes de crearla? ¿Se usó `execute_sql` para el DDL (nunca `apply_migration`)? ¿Fechas declaradas como TEXT? ¿INSERTs en lotes ≤500?
- [ ] ¿Si se envió correo: asunto con prefijo [XSell P&C], KPIs y hallazgos del dashboard real, tono ejecutivo?
- [ ] ¿El dashboard se entregó como archivo `.html` autocontenido listo para subir al portal XSell?
