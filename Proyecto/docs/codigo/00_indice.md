# Código explicado — ETL Clima Perú

Guía de estudio que recorre, celda por celda, los notebooks de `Proyecto/A_Raw` →
`Proyecto/D_Gold`. Es documentación aparte de los notebooks: no reemplaza el código, lo
explica al lado para poder repasarlo sin tener que abrir Databricks.

Para el *qué* (arquitectura, catálogo de tablas, KPIs) ver [`Proyecto/README.md`](../../README.md).
Para el *por qué* de cada decisión de diseño ver [`Proyecto/DECISIONS.md`](../../DECISIONS.md).
Esto de acá es el *cómo*: qué hace cada línea y qué conceptos de ingeniería de datos ilustra.

## Orden de lectura sugerido

1. [`01_a_raw.md`](01_a_raw.md) — ingesta desde la API de Open-Meteo a JSON crudo (capa Raw).
2. [`02_b_bronze.md`](02_b_bronze.md) — JSON → tablas Delta anchas, sin transformar (capa Bronze).
3. [`03_c_silver.md`](03_c_silver.md) — tipado, limpieza, deduplicado (capa Silver).
4. [`04_d_gold_dimensiones.md`](04_d_gold_dimensiones.md) — `dim_location`, `dim_time`, `dim_weather_code`.
5. [`05_d_gold_fact_kpis.md`](05_d_gold_fact_kpis.md) — `fact_weather_daily` y los 3 KPI marts.

## Mapa mental del pipeline

```
Open-Meteo API (forecast + historical)
        │  requests + reintento con backoff
        ▼
   A_Raw          JSON crudo, un archivo por ubicación/data_type/día
        │  pandas: aplanar hourly/daily a filas
        ▼
   B_Bronze       weather_hourly, weather_daily (Delta, sin tipar)
        │  pandas: cast de tipos, rename, drop_duplicates
        ▼
   C_Silver       weather_hourly, weather_daily (Delta, tipado y limpio)
        │  Spark SQL: CAST + rename a nombres de negocio
        ▼
   D_Gold         dim_location, dim_time, dim_weather_code,
                  fact_weather_daily,
                  kpi_agro_daily, kpi_energy_daily, kpi_climate_risk_daily
        │
        ▼
   Power BI       ver docs/powerbi_dashboard.md
```

## Convenciones que se repiten en todos los notebooks

- **`dbutils.widgets`**: parámetros de entrada de un notebook Databricks. Se leen con
  `dbutils.widgets.get("nombre")` y siempre devuelven texto (`str`); por eso cosas como
  `forecast_days` se convierten con `int(...)` después de leerlas.
- **`%sql` al inicio de una celda**: la celda completa se interpreta como SQL en vez de
  Python (es un "magic command" de Databricks).
- **`DROP TABLE/DATABASE IF EXISTS ... ; CREATE TABLE/DATABASE IF NOT EXISTS ...`**: patrón
  de "recrear desde cero" — cada corrida del notebook reconstruye la tabla completa con el
  esquema declarado explícitamente, en vez de confiar en que Spark infiera el tipo de cada
  columna. Ver `05_d_gold_fact_kpis.md` y [DECISIONS.md #13-15](../../DECISIONS.md) para por
  qué esto importa tanto en este proyecto.
- **`spark.createDataFrame(pandas_df).write.format("delta").option("mergeSchema", "true").mode("overwrite").saveAsTable(...)`**:
  patrón de escritura estándar del proyecto — toma un DataFrame de pandas, lo convierte a
  Spark, y sobrescribe la tabla Delta completa (no hace `append` ni `merge`/upsert).
