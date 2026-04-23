# Brand Guide — 3eriza P&C Dashboards

## Identidad Visual (NO NEGOCIABLE)

### Colores principales
- Fondo general: #FFFFFF (blanco puro)
- Color principal: #FF6B00 (naranja 3eriza)
- Texto principal: #2d3436
- Texto secundario: #636e72
- Boton Descargar Excel: #00B894 (verde)
- Hover botones: #e65c00

### Colores de estado
- Excelente: #00b894
- Bueno: #0984e3
- Regular: #fdcb6e
- Critico: #d63031

### Tipografia
- Font family: Segoe UI, system-ui, sans-serif
- KPI valor: 2rem bold
- KPI label: 0.75rem uppercase

## Logo 3eriza

### Especificacion
- Texto "3" y "eriza": color #1f1410 (marron casi negro)
- Interior del "3": color #f5ab00 (dorado)
- Siempre usar el vector original como SVG base64
- NUNCA usar caja naranja con "3e" ni sustituto de texto plano

### Dimensiones en dashboards
- Header: height 45px
- Pantalla login: height 60px centrado

## Componentes HTML

### Header
- Logo a la izquierda, titulo centrado/derecha
- Titulo: "Planeamiento y Control — [Cliente]"
- Boton "Descargar Excel": #00B894, alineado a la derecha
- Subtitulo con rango de fechas

### Tarjetas KPI
- Borde superior: 4px solid #FF6B00
- Sombra: 0 2px 8px rgba(0,0,0,0.08)
- Padding: 20px, border-radius: 8px
- Valor: 2rem bold, color segun estado
- Label: 0.75rem uppercase, color #636e72

### Graficos (Chart.js)
- Color primario: #FF6B00
- Color secundario: #0984e3
- Color terciario: #00b894
- Grids: rgba(0,0,0,0.05)
- Tooltip: rgba(45,52,54,0.9)

### Tabla de datos
- Header: background #FF6B00, texto blanco
- Filas alternas: #fff / #fff8f5
- Hover: #fff3e0
- Border: 1px solid #e0e0e0

## Estructura HTML Obligatoria
- Header: logo + titulo + boton Excel
- Seccion filtros: fecha, campana, tipo resultado
- Grid KPI cards (4-6 cards)
- Graficos principales: barras, lineas, dona
- Tabla detallada con paginacion
- Footer: timestamp ultima actualizacion

## Validacion Final (checklist)
1. Accesos a columnas en minusculas (r.tipo_resultado, NO r.TIPO_RESULTADO)
2. Sin comillas tipograficas (U+2018/2019/201C/201D) en bloques script
3. Chart.js importado correctamente
4. dashboard-data usa d.data (NO d.rows)
5. Fechas declaradas como TEXT en DDL Supabase

