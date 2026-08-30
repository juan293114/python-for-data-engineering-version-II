# D_Gold — dimensiones: `dim_location`, `dim_time`, `dim_weather_code`

Notebooks fuente: [`1_gold_dim_location.ipynb`](../../D_Gold/1_gold_dim_location.ipynb),
[`2_gold_dim_time.ipynb`](../../D_Gold/2_gold_dim_time.ipynb),
[`3_gold_dim_weather_code.ipynb`](../../D_Gold/3_gold_dim_weather_code.ipynb).

Gold es donde el pipeline empieza a modelar un **esquema estrella**: tablas de dimensión
(el "quién/cuándo/qué categoría") separadas de la tabla de hechos y los KPIs (el "cuánto").
Estas tres dimensiones no dependen de las mediciones del día a día — se pueden reconstruir
solo con lo que ya existe en Silver (o, en el caso de `dim_location`, ni siquiera eso).

## 1_gold_dim_location

```sql
CREATE DATABASE IF NOT EXISTS workspace.gold_weather
COMMENT 'Capa Gold: dimensiones, hechos y KPIs de clima'
```

Este notebook es el único que crea el **schema** `gold_weather` (los demás notebooks de
Gold asumen que ya existe). Por eso el Job de Databricks tiene que correr
`1_gold_dim_location` antes que cualquier otro notebook de Gold — si `dim_time` corriera
primero, su `CREATE TABLE workspace.gold_weather.dim_time` fallaría porque el schema
`gold_weather` todavía no existiría.

```sql
DROP TABLE IF EXISTS workspace.gold_weather.dim_location;

CREATE TABLE IF NOT EXISTS workspace.gold_weather.dim_location (
  location_id BIGINT,
  region STRING,
  capital STRING,
  latitude DOUBLE,
  longitude DOUBLE,
  macroregion STRING
)
```

Declarar el `CREATE TABLE` con tipos explícitos (en vez de dejar que Spark los infiera del
DataFrame, como hace Bronze) es lo que le da a Gold un contrato de esquema estable —
cualquier cosa que se conecte después (Power BI incluido) puede confiar en que
`location_id` siempre es `BIGINT`, sin sorpresas.

```python
dbutils.widgets.text("locations_csv_path", "../config/locations.csv", "Locations CSV")
locations_csv_path = dbutils.widgets.get("locations_csv_path")

dim = pd.read_csv(locations_csv_path)[
    ["location_id", "region", "capital", "latitude", "longitude", "macroregion"]
]

df_spark = spark.createDataFrame(dim)
df_spark.write.format("delta") \
    .option("mergeSchema", "true") \
    .mode("overwrite") \
    .saveAsTable("workspace.gold_weather.dim_location")
```

`dim_location` no lee de Silver — lee directo del CSV semilla `config/locations.csv` (el
mismo que usa `A_Raw` para saber a qué coordenadas llamar). Es la única tabla de Gold que no
depende de que haya corrido la ingesta: se puede reconstruir en cualquier momento solo con
el CSV. El `[[...]]` después de `pd.read_csv(...)` selecciona y ordena explícitamente las
columnas que van a la tabla, ignorando cualquier columna extra que el CSV pudiera tener.

## 2_gold_dim_time

```sql
CREATE TABLE IF NOT EXISTS workspace.gold_weather.dim_time (
  date_key BIGINT,
  date DATE,
  year BIGINT,
  quarter BIGINT,
  month BIGINT,
  month_name STRING,
  day BIGINT,
  day_name STRING,
  week_of_year BIGINT,
  season STRING
)
```

Nota sobre los tipos: casi todo acá es `BIGINT` aunque conceptualmente `year` o `month` son
números chicos que cabrían en un `INT`. No es casualidad — es la lección más cara que dejó
este notebook en producción, documentada en
[DECISIONS.md #13, #14 y #15](../../DECISIONS.md): Databricks Serverless (que corre sobre
Spark Connect) sube **cualquier** entero que venga de pandas a `LongType` (`BIGINT`), sin
importar el dtype real de la columna en pandas. Declarar `INT` ahí garantizaba un choque de
tipos (`DELTA_FAILED_TO_MERGE_FIELDS`) cada vez que se recreaba la tabla.

```python
dates = spark.table("workspace.silver_weather.weather_daily").select("observation_date").toPandas()
date_series = pd.to_datetime(dates["observation_date"]).drop_duplicates().sort_values()

dim = pd.DataFrame({"date": date_series})
dim["date_key"] = dim["date"].dt.strftime("%Y%m%d").astype(int)
dim["year"] = dim["date"].dt.year
dim["quarter"] = dim["date"].dt.quarter
dim["month"] = dim["date"].dt.month
dim["month_name"] = dim["date"].dt.month_name()
dim["day"] = dim["date"].dt.day
dim["day_name"] = dim["date"].dt.day_name()
dim["week_of_year"] = dim["date"].dt.isocalendar().week.astype(int)
```

Esta es la técnica clásica para construir una "dimensión de tiempo": se parte de la lista de
fechas que realmente aparecen en los datos (`weather_daily`, sin duplicados), y se deriva
todo lo demás con los accessors `.dt.*` de pandas sobre esa misma columna `date`. Ninguna de
estas columnas viene de la API — todas son cálculo puro de calendario a partir de la fecha.

`date_key` (`YYYYMMDD` como entero, ej. `20260830`) es la convención típica de clave
sustituta para una dimensión de tiempo en modelado dimensional: ordena bien numéricamente y
es fácil de usar como llave de unión (`fact.date_key = dim_time.date_key`) sin tener que
comparar fechas directamente.

```python
season_by_month = {
    12: "verano", 1: "verano", 2: "verano",
    3: "otono", 4: "otono", 5: "otono",
    6: "invierno", 7: "invierno", 8: "invierno",
    9: "primavera", 10: "primavera", 11: "primavera",
}
dim["season"] = dim["month"].map(season_by_month)
```

`.map(diccionario)` reemplaza cada valor de `month` por lo que diga el diccionario en esa
clave — una forma compacta de hacer un "CASE WHEN" en pandas. Las estaciones están definidas
por mes calendario completo (aproximación meteorológica), no por el día exacto del
equinoccio/solsticio — suficiente para KPIs agregados, no para un almanaque astronómico.

```python
dim["date"] = dim["date"].dt.date
```

Este es el fix de [DECISIONS.md #13](../../DECISIONS.md): sin esta línea, `date` se queda
en `datetime64` (con hora), Spark la sube a `TIMESTAMP`, y no matchea con el `DATE`
declarado en el `CREATE TABLE` de arriba. `.dt.date` la baja a un objeto `date` de Python
puro, que sí mapea a `DATE`. Se hace **después** de calcular `year`/`month`/etc. porque esas
columnas sí necesitan que `date` esté en `datetime64` para poder usar `.dt.year`, `.dt.month`,
etc. — si se hiciera antes, esos accessors fallarían.

## 3_gold_dim_weather_code

```python
WEATHER_CODES = [
    (0, "Cielo despejado"),
    (1, "Mayormente despejado"),
    ...
    (99, "Tormenta electrica con granizo fuerte"),
]

dim = pd.DataFrame(WEATHER_CODES, columns=["weather_code", "description"])
```

La más simple de las tres: no lee nada de Silver ni de un CSV, es un catálogo estático
escrito a mano en el propio notebook, tomado de la tabla oficial de códigos WMO que usa
Open-Meteo. `weather_code` (que aparece en `fact_weather_daily`) es un número sin
significado por sí solo — esta dimensión es lo que permite mostrar "Llovizna ligera" en vez
de `51` en un reporte o dashboard.
