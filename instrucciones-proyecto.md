# Instrucciones del Proyecto: XSell P&C — Dashboards

## Contexto
Este proyecto es del área de Planeamiento y Control de **XSell** (antes 3eriza — rebrandeado en junio 2026). BPO/Contact Center, Lima Perú. Centraliza todo lo relacionado con dashboards analíticos, integración de datos MySQL → Supabase, Edge Functions, y el skill `dashboard-3eriza`. Responsable: Franz Palma, Gerente de P&C.

---

## Infraestructura

### Fuentes de Datos (MySQL)
- **Servidor Principal**: `intranetpbx.net.pe:33306` — 95+ BDs de clientes (VOX_TRES, VOX_DOS, 3erizacloud, neotel, etc.)
- **Servidor CASTI**: `66.206.10.130:3306` — BD `bd_casti` (asistencias, trabajadores, servicios, sedes)
- Las credenciales están configuradas en las Edge Functions. No es necesario ingresarlas manualmente.

### Supabase
- Project ID: `ejxpojzuovtsplviafbw`
- API Key de acceso a Edge Functions: `3eriza-pc-2026` (se pasa como parámetro `&key=3eriza-pc-2026`)

### Edge Functions Activas
| Función | Versión | Propósito |
|---------|---------|-----------|
| `dashboard-query` | v8 | API principal: search, describe, query, sync_execute, sync_list, sync_delete, sync_auto_config. `dateStrings:true` — fechas MySQL como TEXT (fix 0000-00-00) |
| `dashboard-data` | v2 | Proxy para dashboards HTML. Sin JWT, CORS abierto. Acciones: `tables`, `columns`, `download`, `query`, `values`. Respuesta siempre `{d: {data: [...]}}` — acceder con `(json.d && json.d.data) \|\| []` |
| `deploy-dashboard` | v6 | Publica dashboards HTML en Netlify (uso legacy — dashboards nuevos se entregan como HTML directo) |
| `auto-sync` | v2 | Cron cada hora, refresca tablas con `auto_sync: true` en `sync_registry` |
| `visor-auth` | v6 | Autenticación del VisorSQL. Password en `VISOR_PASSWORD` env var. Sesiones 30 min |
| `foes-hc-proxy` | v8+ | HC FOES desde Google Sheet. Cuenta `Perfil="Ejecutivo de campo"` AND (`D_Activos="ACTIVO"` OR `F_CESE` vacío) |
| `sync-asistencias-casti` | v6 | Sync diario 15:00 UTC. MySQL2 con `dateStrings:true`. Batches 5,000 filas / 500 sub-batches |
| `excel-lookup` | v1 | Búsqueda cruzada desde Excel. Busca coincidencias en Supabase + CASTI + Principal |
| `mysql-connector` | v3 | Discover y query sobre MySQL CASTI (legacy) |
| `mysql-describe` | v2 | Describe tablas MySQL y guarda en Supabase (usa pg_net) |

### VisorSQL — Explorador de Bases de Datos
- **URL**: `https://visor-sql-3eriza.netlify.app` | **Contraseña**: `xsell-pc-2026`
- **Repo GitHub**: `franzfernandops-max/visor-sql-3eriza` (auto-deploy al push a `main`)
- **Versión actual**: v4 — identidad XSell, logo oficial PNG, login SHA-256, multi-servidor (PRINC + CASTI)

### Portal XSell — Destino de los Dashboards
Los dashboards se entregan como archivos `.html` autocontenidos. Los analistas los suben directamente al portal XSell centralizado. **No se hace deploy a Netlify para dashboards nuevos.**

### Tablas en Supabase
**Sincronizadas**: `reclamos_laive`, `chats_laive`, `gestiones_scaniaarg`, `cat_estados_asistencia`, `reclamos_unna`, `lista_scaniarepuesto`
**Sistema**: `sync_registry`, `query_log`, `cargas_log`, `asistencias_dashboard`, `agentes`, `servicios`, `sedes`, `asistencias`, `indicadores_diarios`, `produccion`, `requerimiento_mensual`, `adt_cohortes`

---

## Flujo Obligatorio para Crear un Dashboard

Los analistas piden dashboards en este proyecto. El flujo es SIEMPRE en este orden — nunca saltarse pasos:

### PASO 1 — Entregar el VisorSQL (OBLIGATORIO PRIMERO)
👉 **https://visor-sql-3eriza.netlify.app** (contraseña: `xsell-pc-2026`)

Decirle al analista: "Ábrelo, busca las tablas del cliente, haz clic en Describir y copia el comando sync para pasármelo." Claude espera el comando sync. **NUNCA crear un HTML explorador alternativo. NUNCA continuar sin el comando sync.**

**Si el analista no sabe usar el VisorSQL**, guiarlo paso a paso:
1. Abrir `https://visor-sql-3eriza.netlify.app`
2. Ingresar contraseña: `xsell-pc-2026`
3. Escribir el nombre del cliente en el buscador (ej: "SCANIA", "LAIVE")
4. Hacer clic en **Describir** junto a la tabla que aparezca
5. Copiar el bloque de comandos que aparece y pegarlo en el chat

### PASO 2 — Sync de la tabla
Con el comando sync recibido:

1. **Verificar si la tabla ya existe** en Supabase antes de crearla:
   ```sql
   SELECT to_regclass('public.nombre_tabla') IS NOT NULL AS existe;
   ```
   - `false` → crear normalmente
   - `true` → preguntar al analista: "¿Reemplazar (DROP + CREATE) o agregar datos nuevos?"

2. **Aplicar filtro de período** — NUNCA sincronizar data histórica sin límite:
   - Por defecto: **solo el año en curso** (`WHERE YEAR(fecha) = YEAR(CURDATE())`)
   - Si no hay data del año en curso: últimos 3 meses
   - Si el analista pide histórico: máximo 12 meses, nunca más
   - Comunicar al analista: "Sincronizaré solo datos del año en curso para mantener el dashboard ágil."

3. Crear la tabla con `execute_sql` (**NUNCA `apply_migration`** — lanza `UnauthorizedException`)
4. Todas las fechas como **TEXT** en el DDL (nunca TIMESTAMPTZ — MySQL puede tener `0000-00-00`)
5. Disparar sync vía `pg_net.http_get` dentro de `execute_sql`
6. Verificar en `net._http_response` — polling máximo 10 intentos. Si sigue en null: "El sync está procesando, verificar en 5 min con `SELECT COUNT(*)`."
7. **Validar post-sync**:
   ```sql
   SELECT COUNT(*) FROM tabla;
   SELECT MIN(fecha), MAX(fecha), COUNT(*) FILTER (WHERE fecha IS NULL) AS nulos FROM tabla;
   ```
   Si nulos > 5% del total, alertar al analista antes de continuar.

### PASO 3 — Selector de columnas (OBLIGATORIO ANTES DE CONSTRUIR)
Después del sync y ANTES de construir el dashboard, Claude genera un artefacto interactivo con checkboxes clasificando TODAS las columnas:
- ✅ **Analíticas** — recomendadas, marcadas por defecto (tipo_resultado, duracion, fechas, etc.)
- ⬜ **Opcionales** — pueden agregar valor, desmarcadas
- 🔒 **Datos Personales** — `nombre_usuario`, `nombre_asesor`, DNI, teléfono, correo, RUC. Permitidas para ranking y filtros internos. **Excluir de la descarga Excel.**
- ⚙️ **Operativas** — IDs de sistema, flags técnicos. No graficables.

El analista confirma cuáles incluir. Claude construye el dashboard **SOLO con las columnas aprobadas**. **NUNCA decidir columnas sin esta confirmación.**

### PASO 4 — Construir el dashboard
Solo después de la confirmación de columnas, construir el HTML con identidad XSell.

### PASO 5 — Entregar el HTML
Entregar el archivo `.html` autocontenido al analista. **El analista lo sube al portal XSell.** No se hace deploy a Netlify.

### PASO 6 — Preguntar frecuencia de auto-sync
Siempre preguntar: cada_hora, cada_6h, diario, semanal, off. **NO asumir frecuencia automáticamente.**

---

## Reglas Técnicas Críticas

### NUNCA usar `apply_migration` — siempre `execute_sql` para DDL
`apply_migration` lanza `UnauthorizedException` en este proyecto.

### NUNCA llamar Edge Functions desde bash
El dominio `supabase.co` está bloqueado en bash. Usar siempre `pg_net` desde `execute_sql`:
```sql
-- Disparar sync
SELECT net.http_get(
  url := 'https://ejxpojzuovtsplviafbw.supabase.co/functions/v1/dashboard-query?action=sync_execute&database=BD&table=TABLA&dest=DEST&key=3eriza-pc-2026'
) AS request_id;

-- Verificar resultado (máx 10 polls)
SELECT id, status_code, content::text
FROM net._http_response WHERE id = [request_id];
-- null = procesando, 200 = éxito, 500 = error
```

### SIEMPRE TEXT para fechas en Supabase DDL
MySQL puede tener fechas `0000-00-00`. Declarar todas las columnas de fecha como `TEXT`.

### SIEMPRE filtrar por período al sincronizar
NUNCA traer data histórica sin límite. Por defecto: año en curso. Máximo permitido: 12 meses.

### COLUMNAS SUPABASE SIEMPRE EN MINÚSCULAS EN JAVASCRIPT
Supabase almacena los nombres de columna en minúsculas aunque MySQL los tenga en MAYÚSCULAS:
- ✅ `r.tipo_resultado`, `r.duracion`, `r.nombre_usuario`, `r.campanya`, `r.fecha`
- ❌ NUNCA `r.TIPO_RESULTADO`, `r.DURACION`, `r.NOMBRE_USUARIO`

### NUNCA usar la API REST de Supabase directamente en dashboards HTML
El navegador bloquea headers `apikey`/`Authorization` por política CORS. **Única solución: usar `dashboard-data` v2 como proxy.**

```javascript
const EF = 'https://ejxpojzuovtsplviafbw.supabase.co/functions/v1/dashboard-data';
// Acceso a datos — SIEMPRE esta forma:
const rows = (json.d && json.d.data) || [];
```

### Contratos API exactos (campos — CRÍTICO)

**`dashboard-data?action=tables`**
- `t.supabase_table` → nombre tabla ✅ | NUNCA `t.name` ❌
- `t.total_filas` → conteo ✅ | NUNCA `t.rows` ❌

**`dashboard-data?action=download` y `action=query`**
- Respuesta: `{d: {data: [...]}, total, limit, offset}`
- Acceso: `(json.d && json.d.data) || []` ✅ | NUNCA `json.data` ❌

**`dashboard-query?action=search`**
- `item.db_name` → nombre BD ✅ | NUNCA `item.database` ❌
- `item.table_name` → nombre tabla ✅ | NUNCA `item.table` ❌
- `item.approx_rows` → filas aproximadas ✅

---

## Identidad Visual XSell (NO NEGOCIABLE)

### Paleta de Colores
| Rol | Hex | Uso |
|-----|-----|-----|
| **Fondo página** | `#1A1025` | Fondo de toda la página — SIEMPRE |
| **Fondo tarjetas** | `#1E1535` | Fondo de todas las cards/paneles |
| **Borde tarjetas** | `#2E2050` | `1px solid` uniforme en los 4 lados — SIN borde izquierdo de color |
| **Naranja primario** | `#FF6B13` | CTAs, barras principales, variaciones +, borde resumen ejecutivo |
| **Púrpura primario** | `#5D17A0` | Gradientes de área, embudo, tracks de gauge |
| **Texto principal** | `#FFFFFF` | Valores KPI, títulos |
| **Texto secundario** | `#8A7FA0` | Labels uppercase, subtítulos |
| **Verde positivo** | `#00C896` | Variaciones positivas ↑ |
| **Rojo negativo** | `#E84545` | Variaciones negativas ↓ |

**NUNCA** usar fondo blanco, naranja `#FF6B00` (es el viejo color 3eriza), ni colores fuera de la paleta.

### Tipografía
- **Anton** — impacto/display
- **Sora** — títulos, KPIs, headers
- **Manrope** — cuerpo, etiquetas, datos

Import obligatorio:
```html
<link href="https://fonts.googleapis.com/css2?family=Anton&family=Sora:wght@300;400;600;700;800&family=Manrope:wght@300;400;500;600;700;800&display=swap" rel="stylesheet">
```

### Logo XSell
**REGLA CRÍTICA**: NUNCA construir el logo como SVG. Siempre usar la imagen base64 oficial. El logo está en el SKILL.md (`/mnt/project/SKILL.md`, sección "Logo XSell"). Altura: `40px` en headers.

### Header Obligatorio
```html
<header style="display:flex;align-items:center;justify-content:space-between;
               padding:16px 24px;border-bottom:1px solid #2E2050;margin-bottom:24px;">
  <div style="display:flex;align-items:center;gap:16px;">
    <img src="data:image/png;base64,{LOGO_B64}" alt="XSell" height="40">
    <div>
      <div style="font-family:Sora,sans-serif;font-size:11px;font-weight:600;
                  color:#8A7FA0;text-transform:uppercase;letter-spacing:1px;">
        Planeamiento y Control
      </div>
      <div style="font-family:Sora,sans-serif;font-size:17px;font-weight:700;color:#fff;">
        {TÍTULO DEL DASHBOARD}
      </div>
    </div>
  </div>
  <span style="font-size:12px;color:#8A7FA0;">{FECHA}</span>
</header>
```

---

## Plantillas de Dashboard (5 tipos)

1. **Inbound** — SLA, Abandono, TMO, FCR, Llamadas. **Sin SPD/HPD.**
2. **Outbound** — Contactabilidad, RPC, Conversión, SPD, TMO.
3. **Encuestas** — Completadas, Tasa Respuesta, Completitud, HPD, TMO.
4. **Perfilamiento** — Leads Perfilados, Penetración, Contactabilidad, HPD, TMO.
5. **Reclamos** — Total, Tasa Atención, Contacto Efectivo, Abandono, HPD, TMO.

### TMO (obligatorio en todos los tipos)
`TMO = solo tiempo en llamada (campo TALK)` — **NUNCA incluir ACW**. Formato `MM:SS`.

### SPD/HPD (Outbound, Encuestas, Perfilamiento, Reclamos — NO Inbound)
`SPD/HPD = Hits del día ÷ (Horas logueado ÷ Horas turno estándar)`
- Solo al cierre del día, NO en tiempo real
- Meta **configurable por campaña** — NUNCA hardcodear

---

## Dashboards Ya Construidos
- **LAIVE** (Reclamos + Chats WSP) — `reclamos_laive`, `chats_laive`
- **SCANIA Argentina** (Outbound) — `gestiones_scaniaarg`
- **UNNA** (Reclamos) — `reclamos_unna`
- **SCANIA Repuesto** (Outbound) — `lista_scaniarepuesto`
- **Asistencias CASTI** — `asistencias_dashboard` + tablas CASTI
- **FOES** — `https://xsell-foes.netlify.app`
- **Protecta** — tabla `protecta_kpi`
- **Calidad Portabilidad Móvil** — `https://dashboard-3eriza-calidad-movil-d9b8b898.netlify.app`
- **Calidad Portabilidad Fija** — `https://dashboard-3eriza-calidad-fija.netlify.app`

---

## Skill

El skill `dashboard-3eriza` está instalado en `/mnt/project/SKILL.md` con la especificación completa:
- Paleta XSell, CSS base, gradientes Chart.js, header, estructura de secciones
- API completa de `dashboard-data` v2 con ejemplos
- Flujos de sync, filtros client-side y server-side, skeleton loading
- Flujo B — búsqueda desde Excel (`excel-lookup` EF)
- Exploración post-dashboard y menú del analista

---

## Roadmap Pendiente
1. CI/CD para app-3eriza (React source + 29 Netlify Functions pendientes de push a repo)
2. Portal sidebar fix — requiere React source
3. VisorSQL session token en todos los dashboards
4. Alertas automáticas por umbrales
5. Dashboards para Inbound, Encuestas y Perfilamiento (pendientes)
6. Vista Asesor como componente reutilizable
