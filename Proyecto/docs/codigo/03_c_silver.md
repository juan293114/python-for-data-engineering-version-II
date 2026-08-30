# C_Silver/1_silver — tipado, limpieza y deduplicado

Notebook fuente: [`Proyecto/C_Silver/1_silver.ipynb`](../../C_Silver/1_silver.ipynb).

Objetivo de esta capa: tomar las tablas Bronze (que traen los tipos "que Spark adivinó") y
dejarlas con tipos explícitos, nombres de columna consistentes, y sin filas duplicadas.
Silver es la última capa genérica — todavía no tiene conceptos de negocio (KPIs, nombres
como `frost_risk`), eso empieza en Gold.

## Celda 1 — imports

```python
import pandas as pd
```

Solo pandas — igual que Bronze, esta capa tampoco toca la red ni archivos sueltos, solo lee
y escribe tablas.

## Celdas 2 y 3 — recrear el esquema Silver

```sql
DROP DATABASE IF EXISTS workspace.silver_weather CASCADE;
```
```sql
CREATE DATABASE IF NOT EXISTS workspace.silver_weather
COMMENT 'Capa Silver: clima tipado y limpio'
```

Mismo patrón "recrear desde cero" que Bronze: cada corrida reconstruye Silver completo a
partir de lo que haya en Bronze en ese momento.

## Celda 4 — `weather_hourly`

```python
HOURLY_TYPES = {
    "location_id": "int64",
    "temperature_2m": "float64",
    ...
    "weather_code": "int64",
}

hourly = spark.table("workspace.bronze_weather.weather_hourly").toPandas()
hourly = hourly.rename(columns={"time": "observation_time"})
hourly["observation_time"] = pd.to_datetime(hourly["observation_time"])
hourly = hourly.astype(HOURLY_TYPES)
hourly = hourly.drop_duplicates(subset=["location_id", "observation_time", "data_type"])

df_spark = spark.createDataFrame(hourly)
df_spark.write.format("delta") \
    .option("mergeSchema", "true") \
    .mode("overwrite") \
    .saveAsTable("workspace.silver_weather.weather_hourly")
```

Paso a paso:

1. **`HOURLY_TYPES`**: un diccionario `{columna: tipo}` que se usa después con
   `.astype(...)`. Tenerlo como diccionario aparte (en vez de castear columna por columna)
   hace explícito, de un vistazo, el contrato de tipos completo de la tabla.
2. **`.toPandas()`**: trae la tabla completa de Spark (distribuida) a un solo DataFrame de
   pandas (en memoria, en el driver). Funciona bien acá porque el volumen es chico (~230k
   filas); con datos mucho más grandes esta transformación se haría en Spark directamente,
   sin pasar por pandas.
3. **`.rename(columns={"time": "observation_time"})`**: Bronze conserva el nombre `"time"`
   tal como lo llama la API; Silver ya empieza a usar nombres más descriptivos.
4. **`pd.to_datetime(...)`**: convierte el string ISO8601 (`"2026-08-30T14:00"`) a un tipo
   fecha/hora real de pandas, para poder operar con fechas más adelante (en Gold).
5. **`.astype(HOURLY_TYPES)`**: fuerza cada columna al tipo declarado. Si alguna columna
   viniera con un tipo incompatible (por ejemplo, texto donde se espera número), esta línea
   lanzaría el error ahí mismo, en vez de dejar el dato mal tipado pasar silenciosamente a
   capas siguientes.
6. **`.drop_duplicates(subset=[...])`**: por si la ingesta se corrió más de una vez para el
   mismo día/ubicación (recordar que `A_Raw` es idempotente por archivo, pero si se borra un
   archivo y se vuelve a descargar, o si el histórico se solapa con el pronóstico en algún
   borde), esto asegura una sola fila por `(ubicación, momento, tipo de dato)`.
7. Escritura: mismo patrón Delta `overwrite` + `mergeSchema` de siempre.

## Celda 5 — `weather_daily`

Mismo patrón que la celda 4, con una diferencia importante:

```python
daily["observation_date"] = pd.to_datetime(daily["time"]).dt.date
```

Acá se usa `.dt.date` (no solo `pd.to_datetime`) para quedarse con un objeto `date` de
Python puro, sin componente de hora. Es más que estilo: si esta línea usara solo
`pd.to_datetime(...)` sin `.dt.date`, la columna quedaría en `datetime64` (con hora), Spark
la mapearía a `TIMESTAMP`, y — si en algún momento se declarara esta columna con un tipo
`DATE` fijo en un `CREATE TABLE` — reventaría con el mismo error que se documenta en
[DECISIONS.md #13](../../DECISIONS.md) para `dim_time`. Como Silver no declara un
`CREATE TABLE` fijo (deja que Spark infiera el esquema en el primer `saveAsTable`, igual que
Bronze), acá no explota, pero **si más adelante se le agrega un `CREATE TABLE` a Silver, hay
que recordar esto**.

`DAILY_TYPES` no incluye `"observation_date"` porque ya quedó como `date` de Python después
del paso anterior, no necesita `.astype`.

`.drop_duplicates(subset=["location_id", "observation_date", "data_type"])` — mismo criterio
que en hourly, pero a grano diario en vez de por timestamp.
