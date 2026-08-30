# Decisiones de diseño — ETL Clima Perú

Registro de por qué el proyecto quedó armado como quedó. Cada entrada indica si la
decisión la tomó el usuario directamente o si fue una decisión adicional propuesta durante
el diseño (marcada como *extra*, notificada en el chat en el momento en que se tomó).

Formato: contexto → decisión → alternativas descartadas → consecuencias.

---

## 1. Fuente de datos: Open-Meteo (Forecast + Historical Weather API)

**Contexto:** se necesita clima por ubicación en Perú, con pronóstico y con histórico para
poder calcular tendencias y anomalías, no solo el estado del momento.

**Decisión:** usar dos endpoints de Open-Meteo:
- Forecast API (`api.open-meteo.com/v1/forecast`) — hasta 16 días de pronóstico horario y
  diario.
- Historical Weather API (`archive-api.open-meteo.com/v1/archive`) — histórico real desde
  2016 hacia atrás, usado aquí solo para los últimos 365 días como referencia estacional.

**Alternativas descartadas:** usar `past_days` del propio Forecast API (máximo 92 días,
insuficiente para una referencia anual) en vez de la Historical API.

**Consecuencias:** Bronze recibe dos orígenes (`data_type = 'forecast' | 'historical'`) que
conviven en la misma tabla ancha, igual que el patrón de una sola tabla `tvmaze` en
`Modulos/01_ELT`.

---

## 2. Alcance geográfico: 1 punto por región (capital), 25 ubicaciones

**Contexto:** el usuario pidió originalmente "grid de puntos por departamento" para más
cobertura espacial, pero eso dispara el volumen de llamadas a la API y la complejidad de
orquestación para un proyecto que primero se prueba en local.

**Decisión:** un punto por región (24 departamentos + Provincia Constitucional del Callao =
25 ubicaciones), representado por la capital de cada una. `dim_location` queda preparada
para crecer a más puntos por región más adelante sin cambiar el modelo (solo agregar filas
a `Proyecto/config/locations.csv`).

**Alternativas descartadas:** 3-5 puntos por región (costa/sierra/selva); grid regular fino
por coordenadas — ambas quedan como posible extensión futura, no se descartan del modelo,
solo se posponen.

**Consecuencias:** 25 series de tiempo por variable en vez de cientos; manejable en local
con pandas/DuckDB y dentro de Databricks Free sin necesidad de jobs particionados.

---

## 3. Ventana histórica: últimos 365 días

**Contexto:** se necesita una referencia histórica para comparar el pronóstico contra un
promedio estacional, sin sobrecargar la ingesta inicial del proyecto.

**Decisión:** cargar los últimos 365 días vía Historical Weather API como carga inicial de
`raw`/`bronze`. Cada corrida posterior solo necesita rellenar el día que se cae de la
ventana (proceso incremental), no se ha automatizado todavía esa reposición.

**Alternativas descartadas:** últimos 5 años (más robusto para anomalías, pero mucho mayor
volumen inicial); rango fijo definido a mano.

**Consecuencias:** las normales climáticas por KPI (ver sección 6) se calculan sobre una
muestra de un año, no sobre "normales" de 30 años en sentido estricto de la OMM. Esto se
documenta también en `README.md` para que no se lea como si fuera una normal climatológica
oficial.

---

## 4. Dominio de KPIs en Gold: agro + energía renovable + riesgo climático

**Contexto:** el usuario eligió explícitamente estos tres dominios de negocio para la capa
Gold (no un dashboard genérico).

**Decisión:** tres tablas Gold de KPIs, una por dominio, todas a grano diario por ubicación:
`gold.kpi_agro_daily`, `gold.kpi_energy_daily`, `gold.kpi_climate_risk_daily`. Todas se
apoyan en la misma tabla base `gold.fact_weather_daily` (grano día × ubicación) para no
recalcular agregaciones tres veces.

**Consecuencias:** ver catálogo completo de KPIs, fórmulas y supuestos en `README.md`.

---

## 5. Umbral de helada: 0 °C

**Contexto:** define cuándo `fact`/`kpi` marcan riesgo de helada, usado tanto en el dominio
agro como en riesgo climático.

**Decisión:** helada meteorológica estándar = temperatura mínima del aire a 2 m ≤ 0 °C
(mismo criterio que usa SENAMHI en sus boletines). Se guarda como columna booleana
`frost_risk` en `kpi_agro_daily` y `kpi_climate_risk_daily`.

**Alternativas descartadas:** esquema multinivel (0 °C / 2 °C / 4 °C) por sensibilidad de
cultivo — se deja como extensión futura documentada, no implementada aún.

---

## 6. Supuestos numéricos adicionales en los KPIs — *extra, pendiente de validar con el usuario*

Estos son valores por defecto razonables para poder tener el modelo completo, pero no
fueron pedidos explícitamente y **deben confirmarse o ajustarse** antes de tomarlos como
definitivos:

- **Grados-día de crecimiento (GDD):** temperatura base 10 °C (`GDD = max(0, temp_media - 10)`),
  valor típico para maíz y muchos cultivos de sierra, pero varía por cultivo real.
- **Lluvia fuerte / alerta:** `precipitation_sum >= 20 mm` en 24 h, umbral genérico de
  "lluvia fuerte" en boletines SENAMHI. No diferenciado por región (costa árida vs selva
  húmeda deberían tener umbrales distintos).
- **Ola de calor:** `temp_max >= 30 °C` como valor único para todo el país. Es un supuesto
  débil: 30 °C es extremo en Puno (sierra) pero común en Piura (costa norte). Candidato
  fuerte a revisar apenas se valide el proyecto con el usuario, probablemente con un
  umbral relativo por macrorregión o por percentil histórico de cada ubicación en vez de
  un valor fijo.
- **Potencial eólico:** clasificación de `wind_speed_max` en bandas (`calmo < 12 km/h`,
  `moderado 12-28`, `fuerte 28-50`, `muy fuerte > 50`) inspirada en la escala de Beaufort,
  no en curvas de potencia de un aerogenerador real.

**Por qué se deja así de todos modos:** para no bloquear el resto del pipeline (dimensiones,
ingesta, Bronze/Silver) esperando a definir cada umbral de negocio. Quedan aislados en
`Proyecto/D_Gold` para poder ajustarlos sin tocar capas anteriores.

---

## 7. Stack local: pandas + DuckDB, aislado de producción

**Contexto:** se necesita poder probar la lógica de ingesta/transformación sin depender de
un cluster Databricks, pero sin que ese código local termine confundido con el código que
sí va a producción (Databricks Free).

**Decisión:** carpeta `Proyecto/local/`, completamente ignorada por git (`.gitignore`).
Ahí se prototipa con pandas + DuckDB (DuckDB simula el rol de las tablas Delta vía SQL).
Una vez validada la lógica, se reescribe "en limpio" dentro de `Proyecto/A_Raw` ...
`Proyecto/D_Gold` en el estilo Spark SQL / Databricks del resto del repo. Si algo se probó
en `local/` y no se traslada, se borra: nunca queda como código muerto en el repo.

**Alternativas descartadas:** PySpark local (mismo motor que producción, pero instalar
Spark/Java localmente es más pesado que lo que amerita esta fase de prototipado).

---

## 8. Ubicación del proyecto dentro del repo: `Proyecto/`

**Contexto:** el repo ya tiene `Modulos/01_ELT/` para el proyecto de ejemplo (TVMaze). Este
es un proyecto nuevo y separado.

**Decisión:** todo el proyecto de clima vive bajo `Proyecto/`, no bajo `Modulos/`, según
indicación explícita del usuario. Dentro de `Proyecto/` se replica el mismo patrón de
subcarpetas `A_Raw / B_Bronze / C_Silver / D_Gold` que usa `Modulos/01_ELT/` — *extra*: esa
paridad de nombres no se pidió explícitamente, se decidió así para que cualquiera que ya
conozca `01_ELT` entienda `Proyecto/` sin curva de aprendizaje adicional.

---

## 9. Nombres de esquema en el catálogo: `raw_weather / bronze_weather / silver_weather / gold_weather` — *extra*

**Contexto:** `Modulos/01_ELT` ya usa los esquemas `workspace.raw`, `workspace.bronze`,
`workspace.silver`, `workspace.gold` en el mismo catálogo (`workspace`, el único disponible
en Databricks Free). El usuario pidió explícitamente mantener el mismo tipo de siglas/paths
que usa `01_ELT` en cada capa (ejemplo dado: `/Volumes/workspace/raw/` y
`CREATE DATABASE IF NOT EXISTS workspace.silver`).

**Decisión:** mismo patrón de nombres que `01_ELT`, con el sufijo `_weather` para no chocar
con las tablas de TVMaze si ambos proyectos corren en el mismo workspace de Databricks Free:

| Capa | Tipo | Nombre |
|---|---|---|
| Raw | Volume | `workspace.raw_weather.clima_pe` (`/Volumes/workspace/raw_weather/clima_pe/`) |
| Bronze | Database | `workspace.bronze_weather` |
| Silver | Database | `workspace.silver_weather` |
| Gold | Database | `workspace.gold_weather` |

**Alternativas descartadas:** reusar los esquemas `raw/bronze/silver/gold` tal cual — se
descarta porque un `DROP DATABASE ... CASCADE` de un proyecto borraría las tablas del otro
(ambos notebooks de ejemplo empiezan con ese `DROP`).

---

## 10. Bronze/Silver separan `weather_hourly` de `weather_daily` — *extra*

**Contexto:** al probar la API contra Ayacucho (ver `Proyecto/local/probe_api.py`), se
confirmó que el bloque `daily` de Open-Meteo ya trae los agregados diarios que
`fact_weather_daily` necesita (`temperature_2m_max/min/mean`, `precipitation_sum`,
`wind_speed_10m_max`, `shortwave_radiation_sum`, `et0_fao_evapotranspiration`,
`uv_index_max`, `sunshine_duration`), calculados por Open-Meteo respetando el día local
(`timezone=auto`) de cada ubicación.

**Decisión:** Bronze y Silver guardan dos tablas por capa (`weather_hourly` y
`weather_daily`) en vez de una sola tabla horaria. `gold.fact_weather_daily` se construye
directamente desde `silver_weather.weather_daily` (solo tipado/renombrado), no
re-agregando horas nosotros mismos.

**Alternativas descartadas:** agregar `weather_hourly` a diario dentro de Gold — se
descarta porque reimplementar el corte de "día local" a mano es una fuente de bugs de
timezone que Open-Meteo ya resuelve mejor del lado del servidor.

**Consecuencias:** `weather_hourly` queda disponible para análisis futuros de grano más
fino (ej. hora pico de radiación), pero no es la fuente de los KPIs actuales de Gold.

---

## 11. Sin coautoría de IA en commits

**Contexto:** instrucción explícita del usuario.

**Decisión:** ningún commit de este repositorio debe incluir `Co-Authored-By` ni mención
equivalente a asistencia de IA, en ningún commit, de ningún proyecto del repo. Regla global,
puesta en `CLAUDE.md` en la raíz, no solo aquí.

---

## 12. Limitación conocida: `uv_index` / `uv_index_max` siempre nulo en histórico

**Contexto:** al correr `B_Bronze/1_bronze` en Databricks salió un `FutureWarning` de pandas
en `pd.concat` sobre columnas "empty or all-NA". Se replicó la misma lógica en
`Proyecto/local/` para diagnosticar: `uv_index_max` viene `NULL` en el 100% de las filas
`historical` (confirmado también llamando directo a `archive-api.open-meteo.com`, sin
nuestro código de por medio) y en 0% de las de `forecast`.

**Causa real:** la Historical Weather API de Open-Meteo (basada en reanálisis ERA5) no
calcula índice UV — es un derivado que solo producen los modelos de pronóstico. No depende
de la ubicación ni de la fecha pedida, siempre va a venir nulo para `historical`.

**Por qué no requiere una decisión de diseño:** `uv_index`/`uv_index_max` están tipados
`float64` en Silver (no `int`), así que el `NULL` no rompe el `.astype()`. Ningún KPI de
Gold (`kpi_agro_daily`, `kpi_energy_daily`, `kpi_climate_risk_daily`) usa esta columna, así
que no afecta ningún resultado actual del pipeline.

**Consecuencia:** si en el futuro se agrega un KPI basado en UV, solo va a poder calcularse
para la ventana de pronóstico (16 días), nunca para el histórico — limitación de la fuente
de datos, no del código de este proyecto.

---

## 13. Bug corregido: `DELTA_FAILED_TO_MERGE_FIELDS` en `dim_time`

**Contexto:** `D_Gold/2_gold_dim_time` falló en Databricks con
`[DELTA_FAILED_TO_MERGE_FIELDS] Failed to merge fields 'date' and 'date'`.

**Causa:** la columna `date` se construía con `pd.to_datetime(...)`, que en pandas queda
como `datetime64[ns]`. `spark.createDataFrame` mapea ese tipo a `TIMESTAMP`, pero la tabla
está declarada como `date DATE` — Delta no fusiona `TIMESTAMP` con `DATE` automáticamente
aunque `mergeSchema=true`.

**Fix:** agregar `dim["date"] = dim["date"].dt.date` justo antes de `spark.createDataFrame`,
después de calcular todas las columnas derivadas que sí necesitan `.dt` (año, mes, etc.),
para que la columna quede como `date` de Python (objeto) y Spark la mapee a `DATE`.

**Por qué no pasó en el prototipo local:** `Proyecto/local/04_gold.py` arma `dim_time` con
`generate_series` de DuckDB directamente sobre columnas `DATE`, sin pasar por
`pd.to_datetime`, así que nunca tuvo este problema. El bug era específico de la traducción
a pandas/Spark del notebook de producción.

---

## 14. Bug corregido: mismos `DELTA_FAILED_TO_MERGE_FIELDS`, pero `INT` vs `BIGINT`

**Contexto:** después de arreglar `date`, `dim_time` volvió a fallar en Databricks, esta vez
con `Failed to merge fields 'week_of_year' and 'week_of_year'`. Se auditó el resto de
notebooks de Gold en busca del mismo patrón antes de que el usuario los corriera y
aparecieran uno por uno.

**Causa:** el mismo problema del punto 13, pero entre enteros de distinto ancho. Se probó
localmente con pandas real (no supuesto) qué tipo produce cada transformación:

| Columna | Origen | dtype real (pandas) | Tipo declarado | ¿Coincide? |
|---|---|---|---|---|
| `year` / `quarter` / `month` / `day` | `.dt.year` etc. | `int32` | `INT` | Sí |
| `week_of_year` | `.dt.isocalendar().week.astype(int)` | `int64` | `INT` | **No** |
| `dim_weather_code.weather_code` | lista de tuplas → `pd.DataFrame(...)` | `int64` | `INT` | **No** |
| `fact_weather_daily.weather_code` | pasa directo desde `silver_weather.weather_daily` (que Silver nunca declara con `CREATE TABLE`, así que Spark infirió `BIGINT` desde el `int64` de pandas) | `int64` / `BIGINT` | `INT` | **No** |

**Fix:** en vez de forzar `.astype("int32")` en cada punto (frágil, hay que acordarse en
cada notebook nuevo), se cambió el DDL de esas tres columnas a `BIGINT` — así coincide con
lo que pandas/Spark producen de forma natural en todo el pipeline. `year`/`quarter`/`month`/
`day` se dejaron como `INT` porque ahí sí coinciden con el `int32` real de pandas.

**Consecuencia práctica:** si se agrega una columna entera nueva en cualquier notebook de
Gold, declararla `BIGINT` por defecto salvo que venga de un accessor `.dt.` de pandas
(`year`, `month`, `day`, `quarter`, `hour`, etc.), que sí son `int32`. Evita repetir este
mismo bug en el próximo KPI que se agregue.
