# A_Raw/1_ingest_to_raw — ingesta desde Open-Meteo

Notebook fuente: [`Proyecto/A_Raw/1_ingest_to_raw.ipynb`](../../A_Raw/1_ingest_to_raw.ipynb).

Objetivo de esta capa: llamar a la API de Open-Meteo para las 25 ubicaciones de
`config/locations.csv` y guardar la respuesta **tal cual llega**, sin transformar nada, en
archivos JSON. Es la única celda del pipeline que sabe que existe internet.

## Celda 1 — imports

```python
import json
import time
from datetime import date, timedelta
from pathlib import Path

import pandas as pd
import requests
```

`json` para serializar la respuesta de la API a disco, `time` para los `sleep` entre
llamadas, `pathlib.Path` para construir rutas de forma segura (funciona igual en Linux que
en Windows, a diferencia de concatenar strings con `/` a mano), `pandas` para leer el CSV de
ubicaciones, `requests` para el HTTP hacia la API.

## Celda 2 — widgets (parámetros de entrada)

```python
dbutils.widgets.text("locations_csv_path", "../config/locations.csv", "Locations CSV")
dbutils.widgets.text("output_volume", "/Volumes/workspace/raw_weather/clima_pe", "Output Volume")
dbutils.widgets.text("forecast_days", "16", "Forecast Days")
dbutils.widgets.text("historical_days", "365", "Historical Days")
dbutils.widgets.text("timeout", "30", "Time Out")
```

`dbutils.widgets` es la forma que tiene Databricks de parametrizar un notebook desde afuera
(desde la UI, o desde un Job que le pasa `base_parameters`) sin editar el código. Cada
`.text(nombre, valor_por_defecto, etiqueta)` crea un campo. Si nadie pasa un valor distinto,
se usa el default — por eso el pipeline corre igual la primera vez que se abre el notebook
que dentro de un Job automatizado.

## Celda 3 — leer los widgets y el CSV de ubicaciones

```python
locations_csv_path = dbutils.widgets.get("locations_csv_path")
output_volume = Path(dbutils.widgets.get("output_volume"))
forecast_days = int(dbutils.widgets.get("forecast_days"))
historical_days = int(dbutils.widgets.get("historical_days"))
timeout = int(dbutils.widgets.get("timeout"))

locations = pd.read_csv(locations_csv_path)
```

`dbutils.widgets.get(...)` siempre devuelve `str`, incluso si el widget "parece" un número
— por eso `forecast_days`, `historical_days` y `timeout` se envuelven en `int(...)`. Si se
te olvida ese cast, más adelante `forecast_days` (un string `"16"`) se pasaría tal cual como
parámetro `forecast_days` a la API, y probablemente funcionaría igual (la mayoría de APIs
HTTP aceptan números como texto), pero es más frágil — mejor tipar apenas se lee.

`locations_csv_path` asume que el notebook corre dentro de un **Databricks Git folder**
(antes llamado Repo): ahí el directorio de trabajo es la carpeta del propio notebook, así
que `../config/locations.csv` sube un nivel (de `A_Raw/` a `Proyecto/`) y entra a
`config/`. Si el notebook se sube suelto al workspace (no como Git folder), esta ruta
relativa no va a resolver y hay que pasar el widget con una ruta absoluta.

## Celda 4 — crear el esquema y el Volume de destino

```python
current_catalog = spark.sql("select current_catalog()").first()[0]
catalog = current_catalog
schema = "raw_weather"
volume = "clima_pe"

spark.sql(f"CREATE SCHEMA IF NOT EXISTS {catalog}.{schema}")
spark.sql(f"CREATE VOLUME IF NOT EXISTS {catalog}.{schema}.{volume}")
```

Un **Volume** en Unity Catalog es básicamente una carpeta con gobierno (permisos, catálogo)
sobre almacenamiento de archivos — a diferencia de una tabla, no tiene esquema de columnas,
es para archivos sueltos (acá, JSON). `current_catalog()` detecta el catálogo activo en vez
de asumir el nombre `workspace` a mano, para que el notebook funcione igual si algún día
corre contra otro catálogo. `IF NOT EXISTS` hace que correr esta celda dos veces no falle.

## Celda 5 — variables de la API

```python
HOURLY_VARS = [...]
DAILY_VARS = [...]
```

Listas de nombres de variables tal como los espera el parámetro `hourly=`/`daily=` de
Open-Meteo (separadas por coma al armar la URL más abajo). Tenerlas como listas en vez de
strings largos a mano hace más fácil agregar/quitar una variable sin romper la sintaxis.

## Celda 6 — funciones auxiliares

```python
def fetch(url: str, params: dict, retries: int = 5) -> dict:
    for attempt in range(retries):
        response = requests.get(url, params=params, timeout=timeout)
        if response.status_code == 429:
            wait = int(response.headers.get("Retry-After", 10 * (attempt + 1)))
            time.sleep(wait)
            continue
        response.raise_for_status()
        return response.json()
    response.raise_for_status()
    return response.json()
```

`fetch` centraliza el llamado HTTP con reintento. `429 Too Many Requests` es el código que
devuelve Open-Meteo cuando se le pega muy seguido (pasó en la corrida real de este proyecto
con las 25 ubicaciones seguidas). En vez de dejar que la excepción tumbe todo el notebook,
el `for attempt in range(retries)` reintenta hasta 5 veces, esperando lo que diga el header
`Retry-After` del servidor (o una espera creciente `10 * (attempt + 1)` si el servidor no lo
manda). Si se agotan los reintentos, el último `response.raise_for_status()` sí deja
propagar el error — a propósito: después de 5 intentos, tapar el error sería peor que
fallar visiblemente.

```python
def out_path_for(data_type: str, region: str) -> Path:
    partition_date = date.today().isoformat()
    out_dir = output_volume / data_type / region.replace(" ", "_")
    out_dir.mkdir(parents=True, exist_ok=True)
    return out_dir / f"{partition_date}.json"
```

Construye la ruta de salida siguiendo el patrón de partición
`{data_type}/{region}/{fecha_de_hoy}.json` (p. ej.
`forecast/Huancavelica/2026-08-30.json`). `region.replace(" ", "_")` evita espacios en
nombres de carpeta (algunas regiones como "San Martin" o "Madre de Dios" los tienen).
`mkdir(parents=True, exist_ok=True)` crea toda la ruta de carpetas si no existe, sin fallar
si ya existía (equivalente a `mkdir -p` en la terminal).

```python
def save(payload: dict, out_path: Path) -> Path:
    out_path.write_text(json.dumps(payload), encoding="utf-8")
    return out_path
```

Serializa el diccionario de respuesta de la API a texto JSON y lo escribe en disco.

## Celda 7 — el loop principal

```python
today = date.today()
start_historical = (today - timedelta(days=historical_days)).isoformat()
end_historical = (today - timedelta(days=1)).isoformat()

saved_files = []
for row in locations.itertuples():
    forecast_path = out_path_for("forecast", row.region)
    if not forecast_path.exists():
        forecast = fetch("https://api.open-meteo.com/v1/forecast", {...})
        forecast["location_id"] = row.location_id
        forecast["region"] = row.region
        saved_files.append(save(forecast, forecast_path))
        time.sleep(1.0)

    historical_path = out_path_for("historical", row.region)
    if not historical_path.exists():
        historical = fetch("https://archive-api.open-meteo.com/v1/archive", {...})
        ...
```

Puntos clave:

- **`locations.itertuples()`** recorre el DataFrame de ubicaciones fila por fila como
  tuplas con nombre (`row.region`, `row.latitude`, etc.) — más rápido que `.iterrows()` y
  más legible que indexar por posición.
- **Idempotencia**: `if not forecast_path.exists():` — si el archivo de hoy para esa región
  ya existe, no se vuelve a llamar la API. Esto es lo que permitió, en la corrida real,
  reintentar el script completo después de un `429` sin tener que rehacer las 17
  ubicaciones que ya se habían descargado bien.
- **Dos endpoints distintos**: `api.open-meteo.com/v1/forecast` (pronóstico, hasta
  `forecast_days` días hacia adelante) y `archive-api.open-meteo.com/v1/archive`
  (histórico, entre `start_historical` y `end_historical`, con reanálisis ERA5 — ver
  [DECISIONS.md #12](../../DECISIONS.md) sobre por qué `uv_index` siempre viene nulo ahí).
- **`forecast["location_id"] = row.location_id`**: la respuesta de la API no sabe a qué
  ubicación pertenece (solo trae lat/lon), así que se le inyectan `location_id` y `region`
  antes de guardarla — Bronze los necesita para poder unir con `dim_location` más adelante.
- **`time.sleep(1.0)`** entre cada llamada: throttling preventivo para no volver a pegarle
  a la API tan seguido que dispare otro `429`.

```python
print(f"Archivos nuevos guardados: {len(saved_files)}")
```

Última línea: un resumen simple de cuántos archivos se escribieron en esta corrida (0 si
todo ya estaba descargado).
