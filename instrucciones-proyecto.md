# Instrucciones del Proyecto: 3eriza P&C — Dashboards

## Contexto
Proyecto del area de Planeamiento y Control de 3eriza (BPO/Contact Center, Lima Peru). Centraliza dashboards analiticos, integracion MySQL -> Supabase, Edge Functions, y el skill dashboard-3eriza. Responsable: Franz Palma, Gerente de P&C.

## Infraestructura

### Fuentes de Datos (MySQL)
- Servidor Principal: intranetpbx.net.pe:33306 — 95+ BDs de clientes (VOX_TRES, VOX_DOS, 3erizacloud, neotel, etc.)
- Servidor CASTI: 66.206.10.130:3306 — BD bd_casti (asistencias, trabajadores, servicios, sedes)

### Supabase
- Project ID: ejxpojzuovtsplviafbw
- API Key Edge Functions: clave interna (ver secrets de Supabase)

### Edge Functions Activas
- dashboard-query v8: API principal search/describe/query/sync sobre 95+ BDs
- dashboard-data v1: Proxy descarga Excel
- deploy-dashboard v5: Publica dashboards en Netlify
- auto-sync v2: Cron cada hora, refresca tablas auto_sync=true
- mysql-connector v3: Discover/query MySQL CASTI
- visor-auth: Autenticacion VisorSQL (tokens UUID 8h)

## Flujo Obligatorio para Crear un Dashboard

SIEMPRE en este orden — nunca saltarse pasos:

### PASO 1 — Entregar VisorSQL (OBLIGATORIO PRIMERO)
Link: https://visor-sql-3eriza.netlify.app
El analista busca tablas, hace clic en Describir, copia el comando sync para Claude.
NUNCA recrear un HTML nuevo del explorador. Esperar el comando sync.

### PASO 2 — Sync de la tabla
1. Crear tabla en Supabase con execute_sql (NUNCA apply_migration)
2. Todas las fechas como TEXT en el DDL (nunca TIMESTAMPTZ)
3. Disparar sync via pg_net.http_get dentro de execute_sql
4. Verificar en net._http_response por request_id
5. Confirmar COUNT(*)

### PASO 3 — Selector de columnas (OBLIGATORIO)
Generar artefacto HTML con checkboxes clasificando columnas:
- Analiticas: recomendadas, marcadas por defecto
- Opcionales: pueden agregar valor, desmarcadas
- Contacto/PII: no graficables (telefono, correo, nombre, ruc)
- Operativas: datos tecnicos del sistema
Esperar confirmacion antes de construir. NUNCA decidir columnas sin confirmacion.

### PASO 4 — Construir dashboard
Solo despues de confirmacion de columnas. HTML con identidad 3eriza.

### PASO 5 — Publicar en Netlify
Via Edge Function deploy-dashboard con pg_net.http_post.

### PASO 6 — Frecuencia de auto-sync
Preguntar: cada_hora, cada_6h, diario, semanal, off.

## Reglas Tecnicas Criticas

### NUNCA apply_migration — siempre execute_sql
apply_migration lanza UnauthorizedException en este proyecto.

### NUNCA llamar Edge Functions desde bash
supabase.co bloqueado en bash. Usar pg_net desde execute_sql.

Patron correcto:
SELECT net.http_get(url := 'https://ejxpojzuovtsplviafbw.supabase.co/functions/v1/dashboard-query?action=sync_execute&database=BD&table=TABLA&dest=DEST&key=KEY') AS request_id;

Verificar:
SELECT id, status_code, content::text FROM net._http_response WHERE id = [request_id];
null = procesando, 200 = exito, 500 = error

### SIEMPRE TEXT para fechas en Supabase
MySQL puede tener fechas 0000-00-00. Declarar todas como TEXT en el DDL.

### COLUMNAS SUPABASE SIEMPRE EN MINUSCULAS EN JS
Supabase lowercasea columnas aunque MySQL las tenga en MAYUSCULAS.
Correcto: r.tipo_resultado, r.duracion, r.nombre_usuario
Incorrecto: r.TIPO_RESULTADO, r.DURACION

## Identidad Visual (NO NEGOCIABLE)
- Fondo: blanco #FFFFFF
- Color principal: naranja #FF6B00
- Header: logo 3eriza + Planeamiento y Control
- Boton Descargar Excel: verde #00B894

## Plantillas de Dashboard (5 tipos)
1. Inbound — SLA, Abandono, TMO, FCR, Llamadas. Sin SPD/HPD.
2. Outbound — Contactabilidad, RPC, Conversion, SPD, TMO.
3. Encuestas — Completadas, Tasa Respuesta, Completitud, HPD, TMO.
4. Perfilamiento — Leads Perfilados, Penetracion, Contactabilidad, HPD, TMO.
5. Reclamos — Total, Tasa Atencion, Contacto Efectivo, Abandono, HPD, TMO.

## TMO
TMO = solo tiempo en llamada (campo DURACION o TALK). NUNCA incluir ACW. KPI obligatorio.

## SPD/HPD
Formula: Hits del dia / (Horas logueado / Horas turno estandar). Solo al cierre del dia.
Meta configurable por campana — NUNCA hardcodear. Aplica en Outbound, Encuestas, Perfilamiento, Reclamos. NO en Inbound.

## Contratos de API

### dashboard-query?action=search
- item.db_name -> nombre BD
- item.table_name -> nombre tabla
- item.approx_rows -> filas aproximadas

### dashboard-data?action=tables
- t.supabase_table -> nombre tabla
- t.total_filas -> conteo

### dashboard-data?action=download&table=X
- d.data -> array de objetos (NUNCA d.rows)

## Dashboards Construidos
- LAIVE (Reclamos + Chats WSP): reclamos_laive, chats_laive
- SCANIA Argentina (Outbound): gestiones_scaniaarg
- UNNA (Reclamos): reclamos_unna
- SCANIA Repuesto (Outbound): lista_scaniarepuesto
- Asistencias: asistencias_dashboard + tablas CASTI

## Netlify
- Cuenta: franzfernandops
- VisorSQL: https://visor-sql-3eriza.netlify.app
- Cada dashboard: https://dashboard-3eriza-{cliente}.netlify.app
- Publicacion automatica via deploy-dashboard EF
