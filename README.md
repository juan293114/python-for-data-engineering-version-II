# python-for-data-engineering-version-II
Este repositorio contiene la información para el curso python-for-data-engineering-version-II

FLUJO
┌────────────────────────────────────┐
│             USGS API               │
│                                    │
│ starttime / endtime / GeoJSON      │
└──────────────────┬─────────────────┘
                   │
                   ▼
┌────────────────────────────────────┐
│              RAW                   │
│                                    │
│ /Volumes/workspace/raw/             │
│ usgs_earthquakes/                   │
│                                    │
│ 2023/01/01/*.geojson               │
│ 2023/01/02/*.geojson               │
│ ...                                │
└──────────────────┬─────────────────┘
                   │
                   ▼
┌────────────────────────────────────┐
│             BRONZE                 │
│                                    │
│ workspace.bronze.usgs_earthquakes  │
└──────────────────┬─────────────────┘
                   │
                   ▼
┌────────────────────────────────────┐
│             SILVER                 │
│                                    │
│ workspace.silver.usgs_earthquakes  │
└──────────────────┬─────────────────┘
                   │
                   ▼
┌────────────────────────────────────┐
│              GOLD                  │
│                                    │
│ dim_time                           │
│ dim_location                       │
│ dim_magnitude                      │
│ dim_event_type                     │
│ dim_alert                          │
│ dim_network                        │
│ fact_earthquake                    │
└──────────────────┬─────────────────┘
                   │
                   ▼
             POWER BI
