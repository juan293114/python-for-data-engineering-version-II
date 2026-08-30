# python-for-data-engineering-version-II

Repositorio del curso **Python for Data Engineering II**. Contiene el material de los
módulos del curso (`Modulos/`) y un proyecto propio de punta a punta con arquitectura
medallón sobre Databricks (`Proyecto/`).

## Estructura del repositorio

```
.
├── Modulos/            Material y laboratorios del curso, por módulo
│   ├── 01_Modulo_01/   Programación funcional, async/sync, generadores, decoradores
│   ├── 02_Modulo_2/    Pandas, PyArrow/Polars/Pydantic/Pandera, calidad de datos (Great Expectations)
│   ├── 03_Modulo_3/    Ingestas (API REST, OAuth2, FTP/SFTP, web scraping), Alembic/SQLAlchemy
│   ├── 04_Modulo_4/    Monitoreo, logs y orquestación
│   └── 01_ELT/         Caso práctico guiado: pipeline medallón completo (TVMaze → Databricks)
│
└── Proyecto/            Proyecto propio: ETL de clima para Perú, arquitectura medallón
    ├── README.md         Qué hace el proyecto, arquitectura, catálogo de tablas y KPIs
    ├── DECISIONS.md       Por qué cada decisión de diseño quedó como quedó
    ├── docs/
    │   ├── codigo/        Código explicado celda por celda, para estudiar
    │   └── powerbi_dashboard.md   Qué gráficas armar en Power BI sobre las tablas Gold
    ├── config/            Semilla de datos (25 regiones de Perú con lat/lon)
    ├── A_Raw/             Ingesta cruda desde la API de Open-Meteo
    ├── B_Bronze/          JSON crudo → tablas Delta anchas
    ├── C_Silver/          Tipado, limpieza, deduplicado
    └── D_Gold/            Dimensiones, hecho y KPIs (agro, energía renovable, riesgo climático)
```

## `Modulos/` — material del curso

Notebooks y laboratorios organizados por módulo, cubriendo desde fundamentos de Python
(funcional, async, generadores) hasta ingestas de datos, control de calidad, control de
versiones de esquema (Alembic) y orquestación/monitoreo. `Modulos/01_ELT/` es un caso
práctico aparte: un pipeline medallón completo (`A_Raw → B_Bronze → C_Silver → D_Gold`)
sobre el dataset público de TVMaze, pensado como plantilla de referencia — es el patrón que
sigue `Proyecto/`.

## `Proyecto/` — ETL Clima Perú

Proyecto propio construido sobre ese mismo patrón: trae el clima (pronóstico a 16 días +
histórico de 365 días) de las 25 regiones de Perú desde la API de
[Open-Meteo](https://open-meteo.com/en/docs), y lo convierte en KPIs de negocio en tres
dominios — agricultura, energía renovable y riesgo climático — listos para conectar a
Power BI.

Para el detalle completo (arquitectura, cómo correrlo en local, cómo migrarlo a Databricks,
catálogo de tablas y fórmulas de cada KPI) ver [`Proyecto/README.md`](Proyecto/README.md).
Para entender por qué cada decisión de diseño quedó así (y qué alternativas se descartaron)
ver [`Proyecto/DECISIONS.md`](Proyecto/DECISIONS.md). Para estudiar el código en detalle,
celda por celda, ver [`Proyecto/docs/codigo/`](Proyecto/docs/codigo/00_indice.md).
