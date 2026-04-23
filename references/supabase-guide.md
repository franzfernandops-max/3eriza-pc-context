# Supabase Guide — 3eriza P&C Dashboards

## Infraestructura

### Supabase
- Project ID: ejxpojzuovtsplviafbw
- URL base: https://ejxpojzuovtsplviafbw.supabase.co
- API Key: ver secrets (no hardcodear en HTML/JS frontend)

### Regla critica: NUNCA apply_migration
apply_migration lanza UnauthorizedException en este proyecto.
Usar SIEMPRE execute_sql para todo DDL y operaciones.

### Regla critica: NUNCA llamar EF desde bash
supabase.co bloqueado en bash de Claude. Usar pg_net desde execute_sql.

## Edge Functions Activas

| Funcion | Version | Proposito |
|---------|---------|-----------|
| dashboard-query | v8 | API principal: search, describe, query, sync sobre 95+ BDs |
| dashboard-data | v1 | Proxy descarga Excel desde dashboards HTML |
| deploy-dashboard | v5 | Publica dashboards HTML en Netlify automaticamente |
| auto-sync | v2 | Cron cada hora, refresca tablas con auto_sync=true |
| visor-auth | - | Autenticacion VisorSQL (tokens UUID 8h, rate limit 5/15min) |
| mysql-connector | v3 | Discover/query MySQL servidor CASTI |
| dashboard-viewer | v3 | Visor de dashboards publicados |

## Tablas Supabase

### Datos sincronizados de clientes
- reclamos_laive, chats_laive (LAIVE)
- gestiones_scaniaarg (SCANIA Argentina)
- lista_scaniarepuesto (SCANIA Repuesto)
- reclamos_unna (UNNA)
- asistencias_dashboard, cat_estados_asistencia

### Tablas de sistema
- sync_registry: registro de sincronizaciones programadas
- query_log: log de queries ejecutados
- cargas_log: log de cargas de datos
- visor_rate_limit: control de intentos de login VisorSQL
- dashboards: registro de dashboards publicados
- agentes, servicios, sedes, asistencias

## Patrones pg_net

### GET request
SELECT net.http_get(
  url := 'https://ejxpojzuovtsplviafbw.supabase.co/functions/v1/FUNCION?param=valor&key=KEY'
) AS request_id;

### POST request
SELECT net.http_post(
  'https://ejxpojzuovtsplviafbw.supabase.co/functions/v1/FUNCION',
  jsonb_build_object('campo1', 'valor1', 'campo2', 'valor2'),
  '{"Content-Type":"application/json"}'::jsonb
) AS request_id;

### Verificar respuesta
SELECT id, status_code, content::text
FROM net._http_response WHERE id = [request_id];
-- null = procesando, 200 = exito, 500 = error

## Deploy de Dashboard (deploy-dashboard EF)

Llamada via pg_net POST:
SELECT net.http_post(
  'https://ejxpojzuovtsplviafbw.supabase.co/functions/v1/deploy-dashboard',
  jsonb_build_object(
    'siteName', 'dashboard-3eriza-CLIENTE',
    'htmlContent', 'CONTENIDO_HTML_BASE64_O_STRING',
    'key', 'KEY'
  ),
  '{"Content-Type":"application/json"}'::jsonb
) AS request_id;

Respuesta exitosa: { url: "https://dashboard-3eriza-CLIENTE.netlify.app", siteId: "..." }

## RLS y Seguridad

- Todas las tablas nuevas tienen RLS habilitado automaticamente
  (trigger trg_auto_rls_new_tables)
- 4 politicas service_role creadas automaticamente por tabla
- Edge Functions corren con service_role (bypasan RLS — correcto)
- NUNCA usar TO PUBLIC o TO anon en politicas RLS
- API keys NUNCA en HTML frontend (siempre server-side)

## DDL de Tablas Sincronizadas

Reglas obligatorias al crear tablas para sync:
1. execute_sql (nunca apply_migration)
2. Todas las columnas de fecha como TEXT
3. Primary key autoincremental o uuid

Ejemplo:
CREATE TABLE IF NOT EXISTS mi_tabla_cliente (
  id BIGSERIAL PRIMARY KEY,
  fecha_gestion TEXT,
  tipo_resultado TEXT,
  duracion INTEGER,
  nombre_usuario TEXT,
  campanya TEXT,
  synced_at TIMESTAMPTZ DEFAULT NOW()
);

## Servidores MySQL

### Servidor Principal (intranetpbx.net.pe:33306)
- 95+ bases de datos de clientes
- Edge Function: dashboard-query
- Bases principales: VOX_TRES, VOX_DOS, 3erizacloud, BD_PyC, neotel

### Servidor CASTI (66.206.10.130:3306)
- Edge Function: mysql-connector
- Bases: BD_PyC, Proyectos_IA, 3erizacloud, be_qnext_corp, bd_casti

