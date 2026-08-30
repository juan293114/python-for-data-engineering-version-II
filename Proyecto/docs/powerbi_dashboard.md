# Dashboard en Power BI — guía de construcción

Qué construir en Power BI sobre `workspace.gold_weather`, con las tablas y columnas reales
del proyecto (nada inventado: todo lo de acá existe en Gold hoy mismo). Para los datos de
conexión (server, HTTP path) ver el mensaje de cierre en la conversación o
`Proyecto/README.md`.

## 1. Modelo de datos a armar en Power BI

Al conectar, importa las 7 tablas de `gold_weather` y arma estas relaciones (todas
uno-a-muchos, dimensión → hecho/KPI):

| Desde | Campo | Hacia | Campo |
|---|---|---|---|
| `dim_location` | `location_id` | `fact_weather_daily` / `kpi_agro_daily` / `kpi_energy_daily` / `kpi_climate_risk_daily` | `location_id` |
| `dim_time` | `date_key` | `fact_weather_daily` / los 3 KPI | `date_key` |
| `dim_weather_code` | `weather_code` | `fact_weather_daily` | `weather_code` |

Marca `dim_time[date]` como **tabla de fechas** (Modelado → Marcar como tabla de fechas) —
eso habilita las funciones de time intelligence de DAX (`TOTALYTD`, `SAMEPERIODLASTYEAR`,
etc.) si más adelante se necesitan.

**Filtro importante en cada visual/página**: la columna `data_type` (`'forecast'` |
`'historical'`) convive en la misma tabla — sin filtrarla, un promedio o una suma va a
mezclar pronóstico con histórico. Recomendado: un slicer de `data_type` visible en cada
página, o duplicar páginas (una vista "Pronóstico", otra "Histórico").

**Modo de conexión**: con ~9,525 filas en las tablas más grandes, **Import** (no
DirectQuery) es lo indicado — carga todo a memoria una vez, los visuales responden al
instante, y no dependen de que el SQL warehouse serverless esté despierto (se autosuspende
a los 10 minutos de inactividad; la primera consulta después de eso tarda unos segundos en
"despertarlo"). Programa un refresh diario (o manual) para traer el pronóstico actualizado.

## 2. Medidas DAX sugeridas

```dax
Temp Máx Promedio = AVERAGE(fact_weather_daily[temp_max])
Temp Mín Promedio = AVERAGE(fact_weather_daily[temp_min])
Precipitación Total (mm) = SUM(fact_weather_daily[precipitation_sum])
Días con Helada = CALCULATE(COUNTROWS(kpi_agro_daily), kpi_agro_daily[frost_risk] = TRUE())
Días con Calor Extremo = CALCULATE(COUNTROWS(kpi_climate_risk_daily), kpi_climate_risk_daily[heat_alert] = TRUE())
Déficit de Riego Promedio (mm) = AVERAGE(kpi_agro_daily[irrigation_deficit_mm])
Horas de Sol Promedio = AVERAGE(kpi_energy_daily[sunshine_hours])
```

## 3. Páginas del dashboard

### Página 1 — Resumen general (todas las regiones)

- **Mapa** (visual "Mapa" o "Mapa de formas" con `latitude`/`longitude` de `dim_location`):
  un punto por región, tamaño o color por `Temp Máx Promedio` — da la primera lectura visual
  de "dónde hace más calor/frío en Perú ahora mismo".
- **Tarjetas KPI** (4-5): `Temp Máx Promedio`, `Precipitación Total (mm)`, `Días con Helada`,
  `Días con Calor Extremo` — con el slicer de `data_type` = `forecast` para que muestren "lo
  que se espera los próximos 16 días".
- **Slicer**: `data_type`, `dim_time[season]`, `dim_location[macroregion]` (costa/sierra/selva).
- **Gráfico de líneas**: `temp_max`/`temp_min` por `dim_time[date]`, con `dim_location[region]`
  como leyenda (filtrando a 3-4 regiones representativas, una por macrorregión, para no
  saturar el gráfico de 25 líneas).

### Página 2 — Agro

- **Mapa de calor / mapa relleno por región**: color por `Días con Helada` (o
  `growing_degree_days` promedio) — identifica de un vistazo las regiones de mayor riesgo
  para cultivos sensibles (probablemente sierra alta: Pasco, Huancavelica, Puno).
- **Gráfico de barras horizontales**: `irrigation_deficit_mm` promedio por `region`, ordenado
  descendente — qué regiones necesitarían más riego suplementario si no llueve.
- **Gráfico de columnas apiladas o área**: `growing_degree_days` acumulado por `dim_time[date]`
  para una región seleccionada (con un slicer de región) — curva de acumulación típica de
  un tablero agroclimático.
- **Tabla/matriz**: `region` × `month_name`, valores = `Días con Helada` — calendario de
  riesgo de heladas por mes y región.

### Página 3 — Energía renovable

- **Gráfico de barras**: `solar_potential_mj_m2` promedio por `region`, ordenado
  descendente — ranking de mejores regiones para generación solar.
- **Gráfico de dona o barras apiladas al 100%**: distribución de `wind_power_class`
  (`calmo`/`moderado`/`fuerte`/`muy_fuerte`) — cuántos días caen en cada banda, en total o
  por región.
- **Matriz (heatmap de tabla)**: `region` × `month_name`, valores = `Horas de Sol Promedio`
  — estacionalidad del recurso solar por región.
- **Gráfico de dispersión (scatter)**: `solar_potential_mj_m2` (eje X) vs
  `sunshine_hours` (eje Y), un punto por región — para ver qué tan correlacionados están
  (esperable que sí, es una forma simple de validar los datos a ojo).

### Página 4 — Riesgo climático / alertas

- **Mapa** con burbujas de tamaño = `Días con Calor Extremo` + `Días con Helada` combinados
  (o dos mapas lado a lado, uno por tipo de alerta).
- **Gráfico de barras**: conteo de `heavy_rain_alert = TRUE()` por `region` — regiones con
  más días de lluvia fuerte en el período.
- **Tabla**: los N días con más alertas simultáneas (`frost_alert` + `heat_alert` +
  `heavy_rain_alert`), útil como "tabla de eventos destacados" en vez de solo agregados.
- **Slicer de fecha** (`dim_time[date]`) para poder acotar a una semana/mes específico y ver
  el detalle día a día en la tabla.

### Página 5 (opcional) — Pronóstico vs. histórico

- **Gráfico de líneas combinado**: para una región seleccionada, `temp_max` de los próximos
  16 días (`data_type = 'forecast'`) superpuesto contra `temp_max` del mismo rango de fechas
  hace un año (`data_type = 'historical'`, filtrado por mes/día equivalente) — el tipo de
  comparación "¿este pronóstico es más cálido/frío que lo normal?" que sustenta buena parte
  del valor de negocio de tener las dos fuentes conviviendo en el modelo.

## 4. Notas para cuando se refine el modelo

- Si más adelante se ajustan los umbrales de `heat_alert`/`irrigation_deficit_mm` (ver
  [DECISIONS.md #6](../DECISIONS.md), son supuestos por región validados a medias), los
  visuales de esta guía no cambian de estructura — solo van a reflejar los nuevos umbrales
  automáticamente en el próximo refresh.
- Si se agregan más puntos por región (hoy es 1 punto = la capital, ver
  [DECISIONS.md #2](../DECISIONS.md)), el mapa de la Página 1 va a mostrar más densidad de
  puntos sin cambios adicionales — el modelo ya está pensado para esa extensión.
