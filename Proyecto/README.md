# Simulación de un Proceso ELT con Arquitectura Medallón y Orquestación con Databricks Jobs

****Data Science Research Perú****\
****Programa de Especialización en Data Engineering Multicloud****\  
****Módulo: Python para Ingeniería de Datos****\  
****Grupo 3****\

****Integrantes:****  
Lizbeth Aquisse  
Gabriela Ramos

# Proceso ELT para conexiones de internet fijo

Para procesar información de conexiones de internet fijo y transformarla desde un dataset fuente en Excel hasta un modelo dimensional en la capa Gold se siguió el modelo de ingesta con una arquitectura Medallón.

## Objetivo

Construir un pipeline ELT que nos permita:

-   Realizar la ingesta desde el dataset original.
-   Almacenar los datos crudos.
-   Realizar limpieza y estandarización en la capa Silver.
-   Construir un modelo dimensional en la capa Gold.
-   Preparar los datos para análisis y consultas posteriores con Power BI.

## Arquitectura
```text
Excel Dataset
CONEXIONES_INTERNET_FIJO_DISTRITO.xlsx
              │
              ▼
             RAW
      Databricks Volume
              │
              ▼
        Pandas / Spark
              │
              ▼
           BRONZE
          Delta Lake
       Datos sin limpiar
              │
       Limpieza / ETL
              ▼
           SILVER
     Datos estandarizados
              │
    Modelado dimensional
              ▼
            GOLD
         Star Schema
```

El dataset que hemos utilizado lo sacamos de la plataforma de datos abiertos y estadísticas de Osiptel, [PUNKU](https://punku.osiptel.gob.pe/) (en quechua, puerta o portal), el cual muestra información actualizada sobre el sector nacional de telecomunicaciones. Este archivo se encuentra almacenado en una carpeta .zip que se puede descargar directamente haciendo clic en Datasets PUNKU

El archivo se encuentra originalmente en formato .xlsx, en la carpeta:
 
`2. INTERNET FIJO`

Tras una readaptación de algunos campos sin alterar los datos originales hicimos la subida del documento directamente a Databricks como upload hacia un Catalog, también se le adaptó el nombre para que no caiga en errores de lectura al momento de trabajar con él en los Notebooks de Databricks:

- Nombre original: [`2.3 CONEXIONES DE INTERNET FIJO POR DISTRITO.`xlsx``](https://docs.google.com/spreadsheets/d/1AhjGsYAOE4fTncg-mEUeHCy_ZsP4oYgK/edit?usp=sharing&ouid=107638657741651867394&rtpof=true&sd=true)

Vemos que los encabezados del archivo no permitían leer correctamente el archivo.

- Nombre adaptado: [`CONEXIONES_INTERNET_FIJO_DISTRITO.xlsx`](https://docs.google.com/spreadsheets/d/1tpiF-WAZMuWOoAM3seen1cG_SCtf3SVi/edit?usp=sharing&ouid=107638657741651867394&rtpof=true&sd=true)

Aquí ya tenemos una mejor organización de los campos y encabezados.

El archivo contiene información sobre conexiones de internet fijo a nivel de departamento, provincia y distrito.

### Campos principales

| CAMPO    | DESCRIPCIÓN                                                                                          | 
| ------------ | ---------------------------------------------------------------------------------------------------------- |
| Periodo     | Periodo de medición                                  |
| Empresa      | Empresa proveedora                                  |
| Departamento | Departamento Geográfico                                    |
| Provincia    | Provincia                                   | 
| Distrito     | Distrito                             |
| Tecnología   | Tecnología usada | Cadena       |
| Segmento     | Segmento del Cliente             |
| Conexiones   | Cantidad de conexiones           |


El archivo fue cargado en un ****Databricks Unity Catalog Volume****:

`/Volumes/internet_fijo_elt/raw/archivos_internet_fijo/`

# Capas del pipeline

## 1\. Raw

La capa Raw contiene el archivo fuente original, en esta no se realizaron transformaciones sobre los datos.

Para la lectura del archivo Excel se utilizó Pandas como herramienta principal.

### Flujo

Excel  
  ↓  
Databricks Volume  
  ↓  
Pandas DataFrame

## 2\. Bronze

En la capa Bronze el objetivo es conservar los datos provenientes de la capa Raw con la menor cantidad de transformaciones posible.

### Proceso

1.  Lectura del archivo Excel mediante Pandas.
2.  Selección de la hoja `Dataset`.
3.  Conversión del DataFrame de Pandas a Spark DataFrame.
4.  Guardar como Delta Table.

### Tabla

`internet_fijo_elt.bronze.conexiones_internet_fijo`

### Flujo

Raw Excel  
    ↓  
Pandas DataFrame  
    ↓  
Spark DataFrame  
    ↓  
Delta Table

El uso de Delta Lake nos permite almacenar los datos en formato transaccional sobre Parquet y facilita su posterior procesamiento dentro de Databricks.

## 3\. Silver

En la capa Silver nos encargamos de limpiar los datos, estandarizarlos, validarlos y prepararlos para alimentar la capa Gold.

****Tabla principal:****

`internet_fijo_elt.silver.conexiones_internet_fijo`

### Transformaciones realizadas

#### 3.1 Normalización de espacios

Aplicamos la limpieza de espacios en las columnas de tipo texto utilizando `trim`, esto nos permite evitar que los valores aparentemente iguales sean tratados como categorías diferentes debido a espacios adicionales.

Por ejemplo `"Lima"` , `"Lima "` pasan a representar el mismo valor.

#### 3.2 Estandarización de Segmento

Se detectaron diferencias de capitalización en los valores de la columna `Segmento`. Estos son ejemplos de los valores originales:

Comercial  
COMERCIAL  
Residencial  
RESIDENCIAL

Proseguimos a normalizarlos para mantener una única representación por categoría:

Comercial  
Residencial

Esto es especialmente importante para la construcción posterior de la dimensión `dim_segmento` y también evitamos generar categorías duplicadas en el modelo dimensional.

#### 3.3 Validación de valores nulos

Asimismo realizamos una validación de valores nulos en las columnas del dataset.

#### 3.4 Validación de duplicados

Realizamos una validación de registros duplicados y no se encontraron registros completamente duplicados en el dataset.

#### 3.5 Validación de la métrica `Conexiones`

Verificamos que la columna `Conexiones` contenga valores válidos para representar la cantidad de conexiones.

### Resultado de Silver

Después de realizar las transformaciones y validaciones, concluimos con la capa Silver limpia la cual proporciona un dataset consistente y preparado para el modelado.

Raw  
 ↓  
Bronze  
 ↓  
Limpieza de espacios  
 ↓  
Estandarización de categorías  
 ↓  
Validación de tipos  
 ↓  
Validación de nulos  
 ↓  
Validación de duplicados  
 ↓  
Validación de métricas  
 ↓  
Silver

La capa Silver funciona como punto de control entre los datos crudos y el modelo analítico de Gold.

# 4\. Gold

En nuestra capa Gold tenemos los datos preparados para consumo analítico.

Implementamos un ****Modelo Estrella****, donde separamos las entidades descriptivas en dimensiones y las métricas en una tabla de hechos.

## Modelo dimensional

```text
                    dim_empresa
                         │
                         │
                         ▼
dim_periodo ──────► fact_conexiones ◄────── dim_segmento
                         ▲
                         │
              ┌──────────┴──────────┐
              │                     │
              │                     │
       dim_tecnologia        dim_ubicacion
      
 ```
## Dimensiones

### `dim_empresa`

Contiene los valores únicos correspondientes a las empresas proveedoras.

id\_empresa  
nombre\_empresa

### `dim_periodo`

Contiene la información temporal utilizada para analizar las conexiones.

fecha\_id  
periodo  
año  
mes  
nombre\_mes  
trimestre

### `dim_segmento`

Contiene los segmentos de clientes previamente normalizados en Silver.

id\_segmento  
segmento

### `dim_tecnologia`

Contiene los tipos de tecnología utilizados para brindar las conexiones.

id\_tecnologia  
tecnologia

### `dim_ubicacion`

Contiene la información geográfica.

id\_ubicacion  
departamento  
provincia  
distrito

La dimensión nos permite analizar las conexiones a diferentes niveles geográficos desde departamento hasta distrito.

## Tabla de hechos

### `fact_conexiones`

La tabla de hechos contiene las métricas y las referencias a las dimensiones.

fecha\_id  
id\_empresa  
id\_ubicacion  
id\_tecnologia  
id\_segmento  
conexiones

La tabla de hechos se construye a partir de la información validada de Silver. Primero se agrupan las conexiones por período, empresa, ubicación, tecnología y segmento, sumando la métrica de conexiones. Posteriormente, se realizan `LEFT JOIN` con las cinco dimensiones Gold para obtener las claves correspondientes.

La métrica principal es:

```
conexiones
```

Esta estructura permite realizar análisis como:

-   Conexiones por empresa.
-   Conexiones por periodo.
-   Conexiones por tecnología.
-   Conexiones por segmento.
-   Conexiones por departamento, provincia o distrito.
-   Evolución temporal de las conexiones.
-   Comparación entre empresas y tecnologías.

# Tecnologías

| Tecnología | Uso |
|---|---|
| **Databricks** | Plataforma de procesamiento y almacenamiento |
| **Python** | Desarrollo del pipeline |
| **Pandas** | Lectura inicial del archivo Excel |
| **PySpark** | Procesamiento y transformación de datos |
| **SQL** | Creación y consulta de tablas |
| **Delta Lake** | Almacenamiento de las capas procesadas |

# Flujo ELT

                    EXTRACT  
                       │  
                       ▼  
              Excel / Databricks  
                  Volume RAW  
                       │  
                       ▼  
                    Pandas  
                       │  
                       ▼  
                   BRONZE  
                       │  
                       │  
                   TRANSFORM  
                       │  
                       ▼  
                    SILVER  
               Limpieza y validación  
                       │  
                       ▼  
                     GOLD  
                  Modelo estrella  
                       │  
                       ▼  
                     LOAD  
                       │  
                       ▼  
                Tablas Delta Lake

## Estructura de datos
 ```text
internet_fijo_elt
│
├── raw
│   └── archivos_internet_fijo
│       └── CONEXIONES_INTERNET_FIJO_DISTRITO.xlsx
│
├── bronze
│   └── conexiones_internet_fijo
│
├── silver
│   └── conexiones_internet_fijo
│
└── gold
    ├── dim_empresa
    ├── dim_periodo
    ├── dim_segmento
    ├── dim_tecnologia
    ├── dim_ubicacion
    └── fact_conexiones

 ```
# Resultado

El pipeline nos permite transformar el dataset fuente desde un archivo Excel hasta un modelo dimensional basado en un esquema Estrella para así mantener una separación clara entre las diferentes etapas del procesamiento:

-   Raw: fuente original sin modificaciones.
-   Bronze: datos crudos persistidos como Delta Table.
-   Silver: datos limpios, estandarizados y validados.
-   Gold: modelo dimensional optimizado para análisis.

Nuestro resultado final es un conjunto de Delta Tables preparadas para las consultas analíticas posteriores y la generación de reportes y futuras herramientas de visualización.

## Consideraciones

El dataset utilizado tiene un tamaño manejable para realizar la extracción inicial mediante Pandas. Las transformaciones de negocio y limpieza se mantienen separadas del modelado dimensional para conservar una arquitectura clara y facilitar el mantenimiento del pipeline.

# Detalles de material subido

El procesamiento está orquestado mediante el siguiente Job en Databricks:

`job_internet_fijo_elt`

El cual se compone de la siguiente forma:

-   Cuaderno de `Raw_to_Capa_Bronze`
-   Cuaderno de `Capa_Bronze_to_Silver`
-   Cuadernos correspondientes a la `Capa_Gold`: creación de tablas de dimensión y de hechos.

Todas estas actividades se realizan de forma secuencial y dependen de la finalización exitosa de la tarea anterior.

El presente Job se encuentra programado para ejecutarse diariamente a las 6:00, en la zona horaria de `America/Lima (UTC -05:00)`.

****Se adjunta la ejecución manual del Job como prueba.****

## Visualización en Power BI

La capa Gold fue conectada directamente con Power BI mediante ****Databricks****, las tablas que se emplearon fueron las siguientes:

gold.dim\_periodo  
gold.dim\_empresa  
gold.dim\_segmento  
gold.dim\_tecnologia  
gold.dim\_ubicacion  
gold.fact\_conexiones

El modelo en Power BI utiliza un esquema estrella, con las dimensiones relacionadas con la tabla de hechos `fact_conexiones`.

El dashboard permite visualizar las conexiones de internet fijo mediante indicadores, gráficos y filtros interactivos, incluyendo:

-   Total de conexiones.
-   Total de empresas.
-   Total de ubicaciones.
-   Total de tecnologías.
-   Evolución mensual de conexiones.
-   Distribución de conexiones por segmento.
-   Top 10 empresas por conexiones.
-   Top 10 departamentos por conexiones.
-   Top 10 provincias por conexiones.
-   Conexiones por tecnología.

Los filtros permiten interactuar con la información por período, tecnología, segmento y empresa.

## Notas importantes

-   El archivo fuente original obtenido desde PUNKU (el cual no se incluirá en GitHub) fue readaptado para que cumpla la estructura necesaria compatible para el proyecto y se conserva en un Volume de Unity Catalog.

-   Las capas Bronze, Silver y Gold se almacenan utilizando Delta Lake.

-   La capa Gold está diseñada para el consumo analítico, y sus tablas son consumidas directamente por Power BI.

-   Cualquier dato sensible empleado durante el desarrollo de este proyecto no formará parte de este repositorio.
