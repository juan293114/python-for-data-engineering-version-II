# ETL Clima Perú — Arquitectura Medallón

Pipeline de datos que trae el clima de las 25 regiones de Perú desde la API de
[Open-Meteo](https://open-meteo.com/en/docs) y lo convierte en KPIs de negocio en tres
dominios: **agricultura**, **energía renovable** y **riesgo climático**.

Sigue la misma arquitectura medallón (`Raw → Bronze → Silver → Gold`) que
[`Modulos/01_ELT/`](../Modulos/01_ELT/), pensado para correr primero en local y luego
migrar a Databricks Free.

## Fuente de datos

Dos endpoints de Open-Meteo, sin API key (uso no comercial):

| Endpoint | Uso | Horizonte |
|---|---|---|
| `api.open-meteo.com/v1/forecast` | Pronóstico | Hasta 16 días hacia adelante, horario + diario |
| `archive-api.open-meteo.com/v1/archive` | Histórico | Últimos 365 días, usado como referencia estacional |

Variables consumidas (horarias y diarias): temperatura (max/min/media a 2m), precipitación
(suma), velocidad y ráfagas de viento, radiación solar de onda corta, evapotranspiración
ET₀, índice UV máximo, duración de sol, código de clima WMO.

## Alcance geográfico

Un punto por región (25 en total: 24 departamentos + la Provincia Constitucional del
Callao), representado por su capital. Ver semilla en
[`config/locations.csv`](config/locations.csv). El modelo admite agregar más puntos por
región sin cambiar el esquema — solo se agregan filas a ese CSV.

## Arquitectura

```
Proyecto/
├── local/              # prototipado pandas + DuckDB, gitignored, no es producción
├── config/
│   └── locations.csv   # semilla: 25 regiones de Perú con lat/lon de su capital
├── A_Raw/               # descarga cruda de la API a JSON, particionado por fecha/región
├── B_Bronze/            # JSON -> tabla ancha Delta, tal cual viene de la API
├── C_Silver/            # tipado, limpieza, nombres de columna estandarizados
└── D_Gold/              # dimensiones, hechos y KPIs por dominio de negocio
```

Catálogo Databricks: `workspace` (único disponible en Databricks Free). Cada capa usa el
mismo esquema de siglas que `01_ELT` (`<capa>_weather`), pero con sufijo `_weather` para que
ambos proyectos convivan en el mismo workspace sin pisarse — un `DROP DATABASE ... CASCADE`
en un proyecto no toca las tablas del otro.

| Capa | Tipo | Nombre en Databricks |
|---|---|---|
| Raw | Volume | `workspace.raw_weather.clima_pe` → `/Volumes/workspace/raw_weather/clima_pe/` |
| Bronze | Database | `workspace.bronze_weather` |
| Silver | Database | `workspace.silver_weather` |
| Gold | Database | `workspace.gold_weather` |

### Raw

```sql
CREATE SCHEMA IF NOT EXISTS workspace.raw_weather;
CREATE VOLUME IF NOT EXISTS workspace.raw_weather.clima_pe;
```

JSON crudo de la API, tal cual responde, en
`/Volumes/workspace/raw_weather/clima_pe/{forecast|historical}/{region}/{yyyy}/{mm}/{dd}/`.

### Bronze

```sql
CREATE SCHEMA IF NOT EXISTS workspace.bronze_weather
COMMENT 'Capa Bronze: clima crudo de Open-Meteo procesado';
```

Dos tablas anchas (mismo patrón que `bronze.tvmaze` en 01_ELT, sin transformar tipos ni
nombres todavía): `bronze_weather.weather_hourly` y `bronze_weather.weather_daily`, cada
una con todas las columnas que trae su bloque de la API (`hourly` / `daily`) más:
- `location_id`, `region`
- `data_type` (`forecast` | `historical`)
- `ingested_at`

Se separan hourly/daily porque el bloque `daily` de Open-Meteo ya viene agregado por día
local (`timezone=auto`) y es la fuente directa de `fact_weather_daily` en Gold — no hace
falta que este proyecto re-agregue horas a mano.

### Silver

```sql
CREATE SCHEMA IF NOT EXISTS workspace.silver_weather
COMMENT 'Capa Silver: clima tipado y limpio';
```

`silver_weather.weather_hourly` y `silver_weather.weather_daily` — mismo tipado correcto
(timestamps, floats), nombres de columna estandarizados en snake_case, deduplicado por
`(location_id, time, data_type)`.

### Gold

```sql
CREATE SCHEMA IF NOT EXISTS workspace.gold_weather
COMMENT 'Capa Gold: dimensiones, hechos y KPIs de clima';
```

**Dimensiones**

- `dim_location` — `location_id`, `region`, `capital`, `latitude`, `longitude`, `macroregion`
  (costa / sierra / selva)
- `dim_time` — `date_key`, `date`, `year`, `month`, `month_name`, `day`, `week_of_year`,
  `season` (verano/otoño/invierno/primavera, hemisferio sur)
- `dim_weather_code` — catálogo de códigos WMO que devuelve Open-Meteo, con su descripción

**Hecho base**

- `fact_weather_daily` — grano día × ubicación. Tipado/renombrado desde
  `silver_weather.weather_daily` (no se re-agrega desde `weather_hourly`):
  `temp_max`, `temp_min`, `temp_mean`,
  `precipitation_sum`, `wind_speed_max`,
  `wind_gusts_max`, `shortwave_radiation_sum`, `et0_fao_evapotranspiration`, `uv_index_max`,
  `sunshine_duration`, `data_type`.

**KPIs por dominio** (todos a grano día × ubicación, todos derivados de `fact_weather_daily`)

#### `kpi_agro_daily`

| KPI | Fórmula | Nota |
|---|---|---|
| `frost_risk` | `temp_min <= 0` | Umbral SENAMHI |
| `growing_degree_days` | `max(0, temp_mean - 10)` | Base 10 °C, supuesto genérico |
| `irrigation_deficit_mm` | `max(0, et0_fao_evapotranspiration - precipitation_sum)` | mm de riego necesarios si la lluvia no cubre la evapotranspiración |

#### `kpi_energy_daily`

| KPI | Fórmula | Nota |
|---|---|---|
| `solar_potential_mj_m2` | `shortwave_radiation_sum` | ya viene en MJ/m² desde la API |
| `sunshine_hours` | `sunshine_duration / 3600` | |
| `wind_power_class` | banda por `wind_speed_max` (calmo/moderado/fuerte/muy fuerte) | escala tipo Beaufort, no curva de potencia real |

#### `kpi_climate_risk_daily`

| KPI | Fórmula | Nota |
|---|---|---|
| `frost_alert` | `temp_min <= 0` | mismo criterio que `frost_risk`, vista de "alerta" en vez de "riesgo agro" |
| `heat_alert` | `temp_max >= 30` | umbral único para todo el país, supuesto débil |
| `heavy_rain_alert` | `precipitation_sum >= 20` | mm en 24h |

## Cómo correr en local

1. Crear el entorno una sola vez (no se versiona, `Proyecto/local/` está en `.gitignore`):
   ```
   python -m venv Proyecto/local/.venv
   Proyecto/local/.venv/Scripts/pip install requests pandas duckdb
   ```
2. Correr en orden: `Proyecto/local/01_raw.py` → `02_bronze.py` → `03_silver.py` →
   `04_gold.py`. Cada uno deja su resultado en `Proyecto/local/data/clima_pe.duckdb`
   (JSON crudo en `Proyecto/local/data/raw/`). `01_raw.py` es idempotente: si el archivo
   del día ya existe no vuelve a llamar la API.
3. Esta es la lógica ya validada contra la API real (25 ubicaciones, forecast + histórico)
   que se tradujo a los notebooks de producción. Si se ajusta algo acá, el cambio
   correspondiente debe reflejarse también en `A_Raw` → `D_Gold`, o se pierde al borrar
   `local/`.

## Cómo migrar a Databricks Free

1. Conectar este repositorio como Databricks Repo (así los notebooks pueden leer
   `Proyecto/config/locations.csv` por ruta relativa, ver widget `locations_csv_path` en
   `A_Raw/1_ingest_to_raw.ipynb` y `D_Gold/1_gold_dim_location.ipynb`).
2. Correr los notebooks en orden: `A_Raw/1_ingest_to_raw` → `B_Bronze/1_bronze` →
   `C_Silver/1_silver` → `D_Gold/1_gold_dim_location` ... `7_gold_kpi_climate_risk_daily`.
3. Cada notebook expone sus parámetros vía `dbutils.widgets` (mismo patrón que
   `Modulos/01_ELT`), no hay que tocar código para cambiar rutas o ventanas de fechas.
4. Los esquemas y el volume (`raw_weather.clima_pe`, `bronze_weather`, `silver_weather`,
   `gold_weather`) se crean solos la primera vez que corre cada notebook.
5. Encadenar los notebooks como un Job (uno por capa/tabla, en orden) para correr el
   pipeline completo con un solo trigger, en vez de ejecutarlos a mano cada vez.

## Estado actual

Los tres tracks están implementados y validados de punta a punta: `Proyecto/local/` corrió
completo contra la API real (25 ubicaciones), y los notebooks de `A_Raw` → `D_Gold` también
corrieron completos dentro de un workspace de Databricks Free real (ingesta, bronze, silver
y las 7 tablas Gold), con los mismos conteos de fila que el prototipo local y resultados con
sentido físico (ej. Pasco y Huancavelica concentran los días de helada, Piura los días de
calor).
