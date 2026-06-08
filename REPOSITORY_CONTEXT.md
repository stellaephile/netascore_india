# NetAScore — Repository Context

This document provides a concise orientation for agents working in this repository.

---

## Project Purpose

**NetAScore** (Network Assessment Score Toolbox) computes **bikeability** and **walkability** indices at the street-segment level from road network data. It is developed by the PLUS Mobility Lab at the University of Salzburg and published in peer-reviewed research (Werner et al. 2024; Stutz et al. 2025).

Scores range from 0 (unsuitable) to 1 (well-suited) and are directional (`_ft` = from-to, `_tf` = to-from).

License: **MIT**

---

## Technology Stack

| Layer | Technology |
|---|---|
| Language | Python 3.8+ |
| Spatial database | PostgreSQL 13 + PostGIS 3.2 |
| SQL templating | Jinja2 / JinjaSql |
| Data import | osm2pgsql, ogr2ogr, raster2pgsql |
| Config format | YAML |
| Container | Docker / Docker Compose |
| Key libs | gdal, igraph, pandas, psycopg2, requests, PyYAML |

---

## Directory Structure

```
netascore/
├── generate_index.py       # CLI entry point — runs the full pipeline
├── settings.py             # Dataclasses: GlobalSettings, DbSettings, DbEntitySettings
├── requirements.txt
├── Dockerfile / docker-compose.yml
│
├── core/                   # One module per pipeline step
│   ├── db_step.py          # Abstract base class DbStep
│   ├── import_step.py      # Data import (OSM, GIP, GeoPackage, CSV)
│   ├── optional_step.py    # Optional layers (DEM, noise, buildings, …)
│   ├── network_step.py     # Network topology (edges + nodes)
│   ├── attributes_step.py  # Attribute derivation from OSM/GIP tags
│   ├── index_step.py       # Index computation from mode profiles
│   └── export_step.py      # Export to GeoPackage via ogr2ogr
│
├── toolbox/
│   ├── helper.py           # Logging (4 levels), timing, string utilities
│   └── dbhelper.py         # PostgresConnection class, SQL execution helpers
│
├── sql/
│   ├── templates/          # Jinja2-templated .sql.j2 files
│   └── functions/          # Reusable PostgreSQL function definitions
│
├── examples/               # Ready-to-use settings YAML and mode profile YAML files
│   ├── settings_osm_query.yml
│   ├── settings_osm_file.yml
│   ├── settings_gip.yml
│   ├── profile_bike.yml
│   └── profile_walk.yml
│
├── data/                   # Input/output data (OSM .pbf, DEM .tif, .gpkg outputs)
│
└── docs / markdown
    ├── README.md
    ├── DOCUMENTATION.md    # Full technical reference
    ├── attributes.md       # Attribute/indicator reference (20+ fields)
    ├── settings.md         # Settings YAML reference
    └── docker.md
```

---

## Pipeline Overview

```
generate_index.py
  1. import_step      — fetch/import OSM or GIP data into PostgreSQL
  2. optional_step    — import supplementary layers (DEM, noise, buildings, …)
  3. network_step     — build network_edge + network_node tables
  4. attributes_step  — derive transport attributes via SQL templates
  5. index_step       — load mode profiles (YAML) → generate dynamic SQL → compute indices
  6. export_step      — write results to GeoPackage (.gpkg)
```

CLI usage:
```
python generate_index.py <settings.yml> [--skip STEP …] [--loglevel 1-4]
```

---

## Configuration

**Settings file (YAML)** — passed as CLI argument, sections:
- `global` — SRID, case ID
- `database` — host, port, user, password, db name (supports env vars `DB_USERNAME`, `DB_PASSWORD`)
- `import` — data source type (`osm_query`, `osm_file`, `gip`), location, query params
- `optional` — supplementary data paths (DEM, noise, buildings, crossings, facilities, greenness, water)
- `profiles` — list of mode profile YAML files + optional road-type filter
- `index` — computation options
- `export` — output path/filename (`<case_id>`, `<srid>` placeholders)

**Mode profile (YAML)** — defines indicator weights and value mappings for a transport mode (bike or walk). `index_step.py` compiles these to dynamic PL/pgSQL at runtime.

---

## Data Sources & Formats

| Format | Role |
|---|---|
| `.osm.pbf` | OSM input (downloaded via Overpass API or provided locally) |
| GIP TXT | Austrian road authority data (Austria only) |
| GeoTIFF `.tif` | DEM for gradient calculation |
| GeoPackage `.gpkg` | Vector supplementary layers; also the main output format |
| PostgreSQL tables | Intermediate processing (schemas: `netascore_data`, `netascore_case_<name>`) |

Output layers in the exported `.gpkg`:
- `edge` — line segments with all attributes and index columns
- `node` — junction points

---

## Key Attributes Computed

`access_car`, `access_bicycle`, `access_pedestrian`, `bicycle_infrastructure`, `pedestrian_infrastructure`, `road_category`, `max_speed`, `pavement`, `width`, `gradient`, `buildings`, `greenness`, `noise`, `water`, `designated_route`, `bridge`, `tunnel`, `crossings`, `facilities`

Final index columns: `index_bike_ft`, `index_bike_tf`, `index_walk_ft`, `index_walk_tf`

---

## Key Design Patterns

- **Factory functions** — each core module exposes `create_<step>(settings)` for instantiation.
- **Abstract base class** — `DbStep.run_step(settings)` is the common interface.
- **Jinja2 SQL templates** — SQL in `sql/templates/` is rendered with settings at runtime; avoids string concatenation.
- **Dynamic PL/pgSQL** — `index_step.py` compiles YAML mode profiles into a PostgreSQL function `calculate_index()` at runtime.
- **`on_existing` conflict handling** — tables can be skipped, deleted, or abort the run.

---

## Database Abstraction (`toolbox/dbhelper.py`)

`PostgresConnection` wraps psycopg2 and provides:
- `execute()` / `ex()` — query execution with error handling
- `execute_sql_from_file()` — raw SQL files
- `execute_template_sql_from_file()` — Jinja2-rendered SQL
- `init_extensions_and_schema()` — PostGIS setup
- `handle_conflicting_output_tables()` — conflict resolution

---

## Logging (`toolbox/helper.py`)

Four verbosity levels (set via `--loglevel`):
1. MajorInfo — phase starts/ends
2. Info — step-level messages
3. Detailed — per-operation messages
4. Debug — verbose diagnostics

Task timing via `logBeginTask()` / `logEndTask()`.

---

## Test / Example Data (`data/`)

Pre-loaded regional datasets for rapid testing:
- **Cities:** Bengaluru, Delhi, Mumbai, Ahmedabad (OSM `.pbf` + DEM `.tif`)
- **Outputs:** Pre-computed `.gpkg` files for reference

---

## Remote

```
https://github.com/stellaephile/active_mobility.git  (branch: main)
```

---

## References

- Werner C. et al. (2024) — Software paper introducing NetAScore
- Stutz D. et al. (2025) — Walkability assessment model (Open Access)
