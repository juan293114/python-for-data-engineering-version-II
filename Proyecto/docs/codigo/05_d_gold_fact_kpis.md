# D_Gold — hecho y KPIs: `fact_weather_daily` + 3 KPI marts

Notebooks fuente: [`4_gold_fact_weather_daily.ipynb`](../../D_Gold/4_gold_fact_weather_daily.ipynb),
[`5_gold_kpi_agro_daily.ipynb`](../../D_Gold/5_gold_kpi_agro_daily.ipynb),
[`6_gold_kpi_energy_daily.ipynb`](../../D_Gold/6_gold_kpi_energy_daily.ipynb),
[`7_gold_kpi_climate_risk_daily.ipynb`](../../D_Gold/7_gold_kpi_climate_risk_daily.ipynb).

Estos cuatro notebooks son distintos a los anteriores en una cosa: **no usan pandas**. Todo
se hace con `spark.sql(...)`, directo sobre las tablas ya existentes. Es la forma más
eficiente de transformar datos que ya están en Spark: evita el viaje de ida y vuelta por
`toPandas()`/`createDataFrame()` cuando la lógica es una simple proyección/agregación que
SQL ya resuelve bien.

## 4_gold_fact_weather_daily

```sql
CREATE TABLE IF NOT EXISTS workspace.gold_weather.fact_weather_daily (
  location_id BIGINT,
  date_key BIGINT,
  data_type STRING,
  weather_code BIGINT,
  temp_max DOUBLE,
  temp_min DOUBLE,
  temp_mean DOUBLE,
  precipitation_sum DOUBLE,
  wind_speed_max DOUBLE,
  wind_gusts_max DOUBLE,
  shortwave_radiation_sum DOUBLE,
  et0_fao_evapotranspiration DOUBLE,
  uv_index_max DOUBLE,
  sunshine_duration DOUBLE
)
```

Esta es la **tabla de hechos**: grano día × ubicación × tipo de dato (pronóstico/histórico),
con las métricas numéricas que interesan para análisis. Todas las tablas de dimensión
(`dim_location`, `dim_time`, `dim_weather_code`) se conectan a esta a través de
`location_id`, `date_key` y `weather_code` respectivamente — ese es el esquema estrella.

```python
fact = spark.sql("""
    SELECT
        location_id,
        CAST(date_format(observation_date, 'yyyyMMdd') AS BIGINT) AS date_key,
        data_type,
        weather_code,
        temperature_2m_max AS temp_max,
        temperature_2m_min AS temp_min,
        temperature_2m_mean AS temp_mean,
        precipitation_sum,
        wind_speed_10m_max AS wind_speed_max,
        wind_gusts_10m_max AS wind_gusts_max,
        shortwave_radiation_sum,
        et0_fao_evapotranspiration,
        uv_index_max,
        sunshine_duration
    FROM workspace.silver_weather.weather_daily
""")
```

Tres cosas pasan en esta única consulta:

1. **`date_format(observation_date, 'yyyyMMdd')` + `CAST(... AS BIGINT)`**: genera el mismo
   `date_key` que calcula `dim_time` (`YYYYMMDD` como entero), pero acá con funciones de
   Spark SQL en vez de pandas — es la llave que después permite unir `fact_weather_daily`
   con `dim_time`.
2. **Renombres con `AS`**: `temperature_2m_max` (nombre "técnico", tal como lo llama la API)
   pasa a `temp_max` (nombre de negocio, más corto y consistente con el resto de columnas de
   esta tabla). Este es el punto donde el pipeline deja de usar los nombres de Open-Meteo y
   empieza a usar los nombres propios del modelo de datos.
3. **No hay ningún `GROUP BY`**: aunque la tabla se llama "daily" y viene de agregar datos
   horarios, esa agregación **ya la hizo Open-Meteo del lado del servidor** (el bloque
   `"daily"` de la API, no `"hourly"`) — ver
   [DECISIONS.md #10](../../DECISIONS.md). Este `SELECT` solo tipa y renombra, no
   re-agrega. Si se re-agregara a mano desde `weather_hourly`, habría que manejar el corte
   de "día local" por zona horaria, algo que Open-Meteo ya resuelve mejor con
   `timezone=auto`.

## 5_gold_kpi_agro_daily

```sql
CREATE TABLE IF NOT EXISTS workspace.gold_weather.kpi_agro_daily (
  location_id BIGINT,
  date_key BIGINT,
  data_type STRING,
  frost_risk BOOLEAN,
  growing_degree_days DOUBLE,
  irrigation_deficit_mm DOUBLE
)
```

```sql
SELECT
    location_id,
    date_key,
    data_type,
    temp_min <= 0 AS frost_risk,
    greatest(0, temp_mean - 10) AS growing_degree_days,
    greatest(0, et0_fao_evapotranspiration - precipitation_sum) AS irrigation_deficit_mm
FROM workspace.gold_weather.fact_weather_daily
```

Tres KPIs, cada uno una expresión de una sola línea sobre `fact_weather_daily`:

- **`frost_risk`**: comparación booleana directa (`temp_min <= 0`) — Spark SQL evalúa esto a
  `true`/`false`, que mapea limpio al `BOOLEAN` declarado.
- **`growing_degree_days`** (grados-día de crecimiento): `greatest(0, temp_mean - 10)` — la
  fórmula estándar de GDD con temperatura base 10°C: si la media del día está por debajo de
  la base, el aporte a "crecimiento acumulado" es 0 (no negativo), por eso el `greatest(0, ...)`.
- **`irrigation_deficit_mm`**: `greatest(0, ET0 - lluvia)` — cuánta agua necesitaría el
  cultivo vía riego si la lluvia del día no alcanza a cubrir la evapotranspiración de
  referencia (ET0). Mismo patrón `greatest(0, ...)` para no mostrar "déficit negativo"
  cuando llueve de más.

Los umbrales `0` (helada) y `10` (base de GDD) son supuestos por defecto sin validar con
negocio real — ver [DECISIONS.md #6](../../DECISIONS.md).

## 6_gold_kpi_energy_daily

```sql
SELECT
    location_id,
    date_key,
    data_type,
    shortwave_radiation_sum AS solar_potential_mj_m2,
    sunshine_duration / 3600.0 AS sunshine_hours,
    CASE
        WHEN wind_speed_max < 12 THEN 'calmo'
        WHEN wind_speed_max < 28 THEN 'moderado'
        WHEN wind_speed_max < 50 THEN 'fuerte'
        ELSE 'muy_fuerte'
    END AS wind_power_class
FROM workspace.gold_weather.fact_weather_daily
```

- **`solar_potential_mj_m2`**: la radiación de onda corta acumulada del día (ya viene en
  MJ/m² desde la API) — a mayor valor, mayor potencial de generación solar ese día.
- **`sunshine_hours`**: `sunshine_duration` viene en segundos desde la API; dividir entre
  `3600.0` (segundos por hora) lo convierte a horas de sol, más legible para un reporte.
- **`wind_power_class`**: un `CASE WHEN` clásico de SQL que clasifica la velocidad máxima de
  viento del día en cuatro bandas categóricas, para poder filtrar/agrupar por "días con
  viento fuerte" sin tener que mirar el número crudo. Las bandas (12/28/50 km/h) están
  inspiradas en la escala de Beaufort, no en la curva de potencia de un aerogenerador real —
  ver [DECISIONS.md #6](../../DECISIONS.md).

## 7_gold_kpi_climate_risk_daily

```sql
SELECT
    location_id,
    date_key,
    data_type,
    temp_min <= 0 AS frost_alert,
    temp_max >= 30 AS heat_alert,
    precipitation_sum >= 20 AS heavy_rain_alert
FROM workspace.gold_weather.fact_weather_daily
```

Tres banderas booleanas, cada una un umbral fijo sobre `fact_weather_daily`:

- **`frost_alert`**: mismo criterio que `frost_risk` de `kpi_agro_daily` (`temp_min <= 0`),
  pero como tabla separada porque conceptualmente es una "alerta de riesgo climático" para
  un consumidor distinto (protección civil) al de `kpi_agro_daily` (decisión agrícola),
  aunque hoy la fórmula sea idéntica.
- **`heat_alert`**: `temp_max >= 30` — el supuesto más débil de todo el modelo: un único
  umbral de 30°C para las 25 regiones no distingue Puno (sierra, frío) de Piura (costa
  norte, calurosa todo el año) — ver [DECISIONS.md #6](../../DECISIONS.md) para el detalle
  de por qué se dejó así de todos modos.
- **`heavy_rain_alert`**: `precipitation_sum >= 20` — 20mm en 24h como umbral genérico de
  "lluvia fuerte" (referencia SENAMHI), tampoco diferenciado por región (costa árida vs.
  selva húmeda deberían tener umbrales distintos en una versión más madura del modelo).

Todos estos umbrales están como literales dentro del `SELECT` (no como constantes en
variables Python, a diferencia de si estuvieran en un notebook de pandas) — son la forma más
simple de expresarlos en Spark SQL puro; si se necesitara parametrizarlos desde afuera,
la vía natural sería convertirlos en widgets del notebook, igual que `forecast_days` en
`A_Raw`.
