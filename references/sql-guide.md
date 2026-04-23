# SQL Guide — 3eriza P&C Dashboards

## Contratos de API (campos exactos)

### dashboard-query?action=search
- item.db_name -> nombre BD (NUNCA item.database)
- item.table_name -> nombre tabla (NUNCA item.table)
- item.approx_rows -> filas aproximadas

### dashboard-query?action=describe
Respuesta: { columns: [...] } con estructura de columnas

### dashboard-query?action=sync_execute
Soporta &where= para filtros. Respuesta asincrona.

### dashboard-query?action=health
Verificacion de conectividad MySQL.

### dashboard-data?action=tables
- t.supabase_table -> nombre tabla (NUNCA t.name)
- t.total_filas -> conteo (NUNCA t.rows)

### dashboard-data?action=download
- d.data -> array de objetos (NUNCA d.rows)

## Patrones SQL BPO

### TMO
TMO = solo tiempo llamada (DURACION o TALK), NUNCA ACW.
SELECT campana, AVG(NULLIF(duracion, 0)) AS tmo_segundos
FROM tabla_llamadas WHERE fecha BETWEEN f1 AND f2 GROUP BY campana;

### SPD/HPD (solo cierre de dia)
Formula: Hits / (Horas_logueado / Horas_turno_estandar)
SELECT agente, COUNT(*) / NULLIF((SUM(tiempo_logueado)/3600.0)/8.0, 0) AS spd
FROM tabla_gestion WHERE DATE(fecha) = fecha_objetivo GROUP BY agente;

### SLA
SELECT campana,
  100.0 * SUM(CASE WHEN tiempo_espera <= 20 THEN 1 ELSE 0 END) / COUNT(*) AS pct_sla
FROM tabla_inbound GROUP BY campana;

### Tasa Abandono
SELECT campana,
  100.0 * SUM(CASE WHEN resultado = 'ABANDONADO' THEN 1 ELSE 0 END) / COUNT(*) AS tasa_abandono
FROM tabla_llamadas GROUP BY campana;

### Contactabilidad Outbound
SELECT campana,
  100.0 * SUM(CASE WHEN tipo_resultado IN ('CONTACTO','RPC') THEN 1 ELSE 0 END) / COUNT(*) AS contactabilidad
FROM tabla_outbound GROUP BY campana;

## Normalizacion de Columnas

### Siempre minusculas en JS (Supabase lowercasea todo)
MySQL: TIPO_RESULTADO -> JS: r.tipo_resultado
MySQL: DURACION -> JS: r.duracion
MySQL: NOMBRE_USUARIO -> JS: r.nombre_usuario
MySQL: CAMPANYA -> JS: r.campanya
MySQL: FECHA_GESTION -> JS: r.fecha_gestion

### Columnas PII — nunca graficar
telefono, celular, correo, email, nombre, apellido, dni, ruc, documento, direccion

### Fechas — siempre TEXT en DDL Supabase
MySQL puede generar 0000-00-00 que rompe TIMESTAMPTZ.
CREATE TABLE mi_tabla (fecha_gestion TEXT, fecha_llamada TEXT);

## Verificacion de Sync

SELECT id, status_code, content::text
FROM net._http_response WHERE id = [request_id];
-- null = procesando, 200 = exito, 500 = error

Para tablas grandes:
SELECT COUNT(*) FROM nombre_tabla_supabase;

