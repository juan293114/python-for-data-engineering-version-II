═══════════════════════════════════════════════════════════════
  PIPELINE DE DATOS METEOROLÓGICOS DEL PERÚ
  Arquitectura Medallón con Databricks y Delta Lake
═══════════════════════════════════════════════════════════════
 
─────────────────────────────────────────────────────────────
¿DE QUÉ TRATA EL PROYECTO?
─────────────────────────────────────────────────────────────
 
Este proyecto construye un pipeline de datos end-to-end que
extrae información meteorológica histórica de las 25 principales
ciudades del Perú — desde Lima hasta Iquitos, desde Tumbes
hasta Puno — y la transforma en un modelo dimensional listo
para análisis y visualización.
 
La fuente es la API pública de Open Meteo, que provee datos
del modelo climático ERA5. No requiere autenticación y tiene
cobertura desde 1940 hasta el día de hoy.
 
Las variables que capturamos son: temperatura, humedad
relativa, velocidad del viento y código de condición
climática por hora.
 
 
─────────────────────────────────────────────────────────────
¿QUÉ ES UNA ARQUITECTURA MEDALLÓN?
─────────────────────────────────────────────────────────────
 
La arquitectura medallón organiza los datos en tres capas
progresivas de calidad:
 
  🥉 RAW / BRONZE → datos tal como llegan de la fuente.
                    Sin tocar, sin transformar.
                    En nuestro caso: JSONs de la API.
 
  ⚪ SILVER       → datos limpios y tipados correctamente.
                    Eliminamos nulos, casteamos tipos,
                    convertimos horarios de UTC a hora local,
                    y decodificamos los códigos climáticos.
 
  🥇 GOLD         → modelo dimensional (Star Schema) listo
                    para consumo en BI o análisis directo.
 
 
─────────────────────────────────────────────────────────────
¿CÓMO FUNCIONA EL PIPELINE? (PASO A PASO)
─────────────────────────────────────────────────────────────
 
PASO 1 — RAW
  Consumimos la API de Open Meteo en un bucle por fechas.
  Por cada día consultado se genera un archivo JSON que se
  guarda en un Volume de Unity Catalog en Databricks.
  Las 25 ciudades se consultan en paralelo en cada request.
 
PASO 2 — BRONZE
  Leemos todos los JSONs del Volume y los cargamos en una
  Delta Table. El esquema usa STRUCT y ARRAY para mantener
  la estructura anidada original de la API intacta.
  Guardamos metadatos de auditoría: fecha de ingesta y
  archivo de origen.
 
PASO 3 — SILVER
  Hacemos "explode" de los arrays horarios para convertir
  cada hora en una fila independiente.
  Aplicamos: cast de tipos, conversión de zona horaria
  (UTC → UTC-5 para Perú), mapeo de weather_code a una
  descripción legible (Despejado / Nublado / Lluvia /
  Otro-Extremo) y eliminación de registros con temperatura
  nula.
 
PASO 4 — GOLD (Dimensiones)
  Construimos tres dimensiones:
 
  · dim_tiempo    → calendario completo con año, mes, día,
                    hora, nombre del día, y flag de fin
                    de semana.
 
  · dim_ubicacion → catálogo de las 25 ciudades con sus
                    coordenadas. Usamos distancia euclidiana
                    para asignar el nombre correcto a cada
                    par latitud/longitud devuelto por la API.
 
  · dim_clima     → catálogo de condiciones climáticas
                    basado en los códigos WMO presentes
                    en los datos.
 
PASO 5 — GOLD (Fact)
  Unimos la tabla Silver con las tres dimensiones mediante
  merges por las claves naturales (timestamp, coordenadas,
  weather_code) y generamos la tabla de hechos final
  con las métricas: temperatura, humedad y velocidad
  del viento.
 
 
─────────────────────────────────────────────────────────────
¿QUÉ TECNOLOGÍAS USAMOS?
─────────────────────────────────────────────────────────────
 
  · Databricks       → plataforma principal, notebooks,
                       clusters y Unity Catalog
 
  · Delta Lake       → formato de almacenamiento con ACID,
                       versionado y time travel
 
  · Apache Spark     → motor de procesamiento distribuido
 
  · Python / Pandas  → transformaciones tabulares
 
  · Open Meteo API   → fuente de datos meteorológicos
                       gratuita y sin autenticación
 
  · Pendulum         → manejo de fechas y zonas horarias
 
 
─────────────────────────────────────────────────────────────
¿QUÉ PODEMOS ANALIZAR CON ESTOS DATOS?
─────────────────────────────────────────────────────────────
 
  · ¿Cuál es la ciudad más caliente del Perú en verano?
  · ¿Qué ciudades tienen mayor humedad promedio?
  · ¿Cuántos días de lluvia tuvo Iquitos vs Lima?
  · ¿A qué hora del día hace más viento en la costa?
  · Comparativa climática entre costa, sierra y selva.
 
 
─────────────────────────────────────────────────────────────
RESULTADO FINAL
─────────────────────────────────────────────────────────────
 
Un modelo estrella en Databricks con:
  · 1 tabla de hechos   → fact_weather
  · 3 dimensiones       → dim_tiempo, dim_ubicacion, dim_clima
  · 25 ciudades         → representando las regiones del Perú
  · Datos horarios      → temperatura, humedad, viento
 
Todo almacenado en Delta Lake dentro del catálogo
"proyecto_final_prueba", listo para conectarse a
Power BI o cualquier herramienta de visualización.
 
═══════════════════════════════════════════════════════════════