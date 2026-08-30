# B_Bronze/1_bronze — JSON crudo a tablas Delta anchas

Notebook fuente: [`Proyecto/B_Bronze/1_bronze.ipynb`](../../B_Bronze/1_bronze.ipynb).

Objetivo de esta capa: leer todos los JSON que dejó `A_Raw` y aplanarlos a dos tablas
Delta — `weather_hourly` y `weather_daily` — sin tipar ni renombrar columnas todavía. Bronze
es "los mismos datos, ahora consultables con SQL", nada más.

## Celda 1 — imports

```python
import json
from pathlib import Path

import pandas as pd
```

Nada de `requests` acá — Bronze nunca toca la red, solo lee archivos que Raw ya dejó en el
Volume.

## Celda 2 — widget de entrada

```python
dbutils.widgets.text("raw_root", "/Volumes/workspace/raw_weather/clima_pe", "Raw Root")
raw_root = Path(dbutils.widgets.get("raw_root"))
```

Mismo Volume que escribió `A_Raw` en su widget `output_volume` — tienen que apuntar al
mismo lugar para que Bronze encuentre los archivos.

## Celda 3 y 4 — recrear el esquema Bronze

```sql
DROP DATABASE IF EXISTS workspace.bronze_weather CASCADE;
```
```sql
CREATE DATABASE IF NOT EXISTS workspace.bronze_weather
COMMENT 'Capa Bronze: clima crudo de Open-Meteo procesado'
```

`DROP ... CASCADE` borra el schema completo con todas sus tablas antes de recrearlo. Esto
significa que **cada corrida de este notebook reconstruye Bronze desde cero** a partir de
todo lo que haya en `raw_root` — no es incremental. Para un proyecto de este tamaño (25
ubicaciones, un año de histórico) es aceptable; en un pipeline con volúmenes mucho más
grandes, esto se cambiaría por una carga incremental (solo los archivos nuevos desde la
última corrida).

## Celda 5 — aplanar los JSON

```python
def load_block(block_name: str) -> pd.DataFrame:
    frames = []
    for data_type_dir in raw_root.iterdir():
        data_type = data_type_dir.name
        for json_file in data_type_dir.glob("*/*.json"):
            payload = json.loads(json_file.read_text(encoding="utf-8"))
            block = pd.DataFrame(payload[block_name])
            block["location_id"] = payload["location_id"]
            block["region"] = payload["region"]
            block["latitude"] = payload["latitude"]
            block["longitude"] = payload["longitude"]
            block["elevation"] = payload["elevation"]
            block["data_type"] = data_type
            frames.append(block)
    return pd.concat(frames, ignore_index=True)
```

Esta función se reutiliza para los dos bloques que trae cada respuesta de Open-Meteo
(`"hourly"` y `"daily"`), por eso recibe `block_name` como parámetro en vez de estar
hardcodeada dos veces.

- `raw_root.iterdir()` recorre las carpetas de primer nivel bajo el Volume: `forecast/` y
  `historical/` — cada una es un `data_type`.
- `data_type_dir.glob("*/*.json")` busca dos niveles más abajo: `{región}/{fecha}.json`.
  El primer `*` es la región, el segundo es el nombre del archivo.
- `payload[block_name]` es un diccionario donde cada variable de la API es una lista
  paralela a `"time"` (ej. `{"time": [...], "temperature_2m": [...], ...}`).
  `pd.DataFrame(payload[block_name])` convierte eso directo a una tabla, una fila por
  timestamp, porque pandas sabe construir un DataFrame a partir de un diccionario de listas
  de igual longitud.
- Las cinco líneas `block["location_id"] = ...` etc. le pegan a cada fila los metadatos que
  la API no devuelve dentro de `hourly`/`daily` (vienen sueltos en el JSON, o los agregó
  `A_Raw` a mano): a qué ubicación pertenece y de qué endpoint vino.
- `pd.concat(frames, ignore_index=True)` junta las ~50 tablas (una por archivo) en una sola.
  `ignore_index=True` reindexa desde 0 en vez de arrastrar los índices originales de cada
  trozo (que si no, se repetirían).

> Esta es la línea donde salió el `FutureWarning` de pandas documentado en
> [DECISIONS.md #12](../../DECISIONS.md): algunos archivos históricos tienen la columna
> `uv_index_max` completamente `NULL` (la API histórica no la calcula), así que al
> concatenar frames con esa columna vacía junto a otros con datos reales, pandas avisa que
> en una versión futura podría inferir el tipo distinto. No afecta el resultado actual.

## Celdas 6 y 7 — escribir las tablas

```python
hourly = load_block("hourly")
hourly["ingested_at"] = pd.Timestamp.utcnow()
df_spark = spark.createDataFrame(hourly)

df_spark.write.format("delta") \
    .option("mergeSchema", "true") \
    .mode("overwrite") \
    .saveAsTable("workspace.bronze_weather.weather_hourly")
```

`ingested_at` es una marca de auditoría — no viene de la API, la agrega el propio proceso
para poder saber después cuándo se cargó cada fila. `spark.createDataFrame(hourly)` sube el
DataFrame de pandas (que vive en el driver) a un DataFrame distribuido de Spark. El patrón
de escritura (`format("delta")`, `mergeSchema=true`, `mode("overwrite")`) es el mismo en
casi todos los notebooks del proyecto — ver la sección de convenciones en
[`00_indice.md`](00_indice.md).

La celda 7 hace exactamente lo mismo para `load_block("daily")` → `weather_daily`. Como
Bronze nunca declara un `CREATE TABLE` con tipos fijos (a diferencia de Silver y Gold), la
primera vez que corre cada `saveAsTable` es la que define el esquema de la tabla, a partir
de lo que infiere `spark.createDataFrame` desde el DataFrame de pandas.
