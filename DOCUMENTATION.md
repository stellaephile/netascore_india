# NetAScore - Complete Technical Documentation

**NetAScore (Network Assessment Score Toolbox)** is an open-source Python framework for computing segment-scale bikeability and walkability indices from road network data. It ingests spatial data from OpenStreetMap (OSM) or the Austrian GIP dataset, enriches each road segment with transport-relevant attributes, applies configurable mode profiles to compute composite routing indices, and exports the final assessed network as a GeoPackage.

---

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [Repository Structure](#2-repository-structure)
3. [Architecture & Data Flow](#3-architecture--data-flow)
4. [Entry Point - `generate_index.py`](#4-entry-point--generate_indexpy)
5. [Settings System - `settings.py`](#5-settings-system--settingspy)
6. [Toolbox](#6-toolbox)
   - [helper.py - Logging & Utilities](#helperpy--logging--utilities)
   - [dbhelper.py - Database Abstraction](#dbhelperpy--database-abstraction)
7. [Core Pipeline Steps](#7-core-pipeline-steps)
   - [Step 1: Import](#step-1-import--core/import_steppy)
   - [Step 2: Optional Imports](#step-2-optional-imports--core/optional_steppy)
   - [Step 3: Network Construction](#step-3-network-construction--core/network_steppy)
   - [Step 4: Attribute Calculation](#step-4-attribute-calculation--core/attributes_steppy)
   - [Step 5: Index Generation](#step-5-index-generation--core/index_steppy)
   - [Step 6: Export](#step-6-export--core/export_steppy)
8. [SQL Layer](#8-sql-layer)
   - [Templates (Jinja2)](#templates-jinja2)
   - [Functions](#functions)
9. [Mode Profiles](#9-mode-profiles)
10. [Configuration Reference](#10-configuration-reference)
11. [Deployment](#11-deployment)
12. [Output Data Model](#12-output-data-model)
13. [Dependencies](#13-dependencies)

---

## 1. Project Overview

NetAScore computes two primary outputs per road segment edge:

- **Bikeability** (`index_bike_ft`, `index_bike_tf`) - suitability score for cycling
- **Walkability** (`index_walk_ft`, `index_walk_tf`) - suitability score for walking

Index values range from `0` (unsuitable) to `1` (well-suited). The `_ft` and `_tf` suffixes indicate travel direction: *from-node to to-node* and *to-node to from-node* respectively, enabling directional routing.

The system is designed to be data-source agnostic and profile-driven. The same pipeline runs on both OSM and GIP data; mode profiles (YAML files) define all scoring logic and can be freely customised without touching the Python or SQL code.

---

## 2. Repository Structure

```
netascore/
├── generate_index.py          # Main entry point - CLI, pipeline orchestration
├── settings.py                # Global/DB settings dataclasses and enums
├── requirements.txt           # Python dependencies
├── Dockerfile                 # Docker image definition
├── docker-compose.yml         # Dev/build compose (build from source)
│
├── core/                      # Pipeline step implementations
│   ├── db_step.py             # Abstract base class for all steps
│   ├── import_step.py         # GIP and OSM data import
│   ├── optional_step.py       # DEM, noise, building, etc. importers
│   ├── network_step.py        # Network graph construction (edge/node tables)
│   ├── attributes_step.py     # Transport attribute computation
│   ├── index_step.py          # Index computation + YAML→SQL profile translation
│   └── export_step.py         # GeoPackage export via ogr2ogr
│
├── toolbox/
│   ├── helper.py              # Logging, timing, YAML-safe string utilities
│   └── dbhelper.py            # PostgresConnection - all DB interaction
│
├── sql/
│   ├── templates/             # Jinja2 SQL templates (.sql.j2)
│   │   ├── gip_network.sql.j2
│   │   ├── osm_network.sql.j2
│   │   ├── gip_attributes.sql.j2
│   │   ├── osm_attributes.sql.j2
│   │   ├── index.sql.j2
│   │   └── export.sql.j2
│   └── functions/             # Reusable SQL functions (.sql and .sql.j2)
│       ├── calculate_index.sql.j2     # Dynamic PL/pgSQL index function
│       ├── determine_utmzone.sql
│       ├── gip_calculate_bicycle_infrastructure.sql
│       ├── gip_calculate_pedestrian_infrastructure.sql
│       ├── gip_calculate_road_category.sql
│       ├── osm_calculate_access_bicycle.sql
│       ├── osm_calculate_access_car.sql
│       ├── osm_calculate_access_pedestrian.sql
│       └── osm_delete_dangling_edges.sql
│
├── resources/
│   └── default.style          # osm2pgsql import style configuration
│
├── examples/                  # Ready-to-use settings and profile files
│   ├── docker-compose.yml
│   ├── settings_gip.yml
│   ├── settings_osm_file.yml
│   ├── settings_osm_query.yml
│   ├── dev_example_*.yml
│   ├── profile_bike.yml
│   └── profile_walk.yml
│
└── docs/
    ├── README.md
    ├── settings.md
    ├── attributes.md
    └── docker.md
```

---

## 3. Architecture & Data Flow

```
Settings YAML file
       │
       ▼
generate_index.py  ←─── CLI args (--skip, --loglevel)
       │
       ├─── 1. IMPORT ──────────────────────────────────────────────────────────
       │         GipImporter           OsmImporter
       │         └─ unzip GIP .txt     └─ download via Overpass API or local file
       │            create_csv/sql        osm2pgsql → osm_point/line/polygon
       │            psql COPY             derive building/crossing/facility/
       │                                            greenness/water tables
       │
       ├─── 2. OPTIONAL IMPORTS ───────────────────────────────────────────────
       │         DemImporter → raster2pgsql → dem table
       │         NoiseImporter, BuildingImporter, ... → ogr2ogr → PostGIS tables
       │
       ├─── 3. NETWORK ────────────────────────────────────────────────────────
       │         gip_network.sql.j2     osm_network.sql.j2
       │         └─ build network_edge  └─ split OSM ways at intersections
       │            network_node           topology correction
       │                                   network_edge, network_node
       │
       ├─── 4. ATTRIBUTES ─────────────────────────────────────────────────────
       │         gip_attributes.sql.j2  osm_attributes.sql.j2
       │         └─ access flags        └─ access flags (car/bike/walk)
       │            bridge/tunnel          bridge, tunnel
       │            road_category          road_category
       │            bicycle_infra          bicycle_infrastructure
       │            pedestrian_infra       pedestrian_infrastructure
       │            speed/gradient/        speed, gradient, pavement,
       │            facilities/noise/...   facilities, noise, greenness...
       │            → network_edge_attributes, network_node_attributes
       │            → network_edge_export (joined view)
       │
       ├─── 5. INDEX ──────────────────────────────────────────────────────────
       │         For each mode profile (bike, walk, ...):
       │           parse profile YAML → generate SQL CASE expressions
       │           register calculate_index() PL/pgSQL function in DB
       │           execute index.sql.j2 → network_edge_index
       │         create export_edge, export_node (joined tables)
       │
       └─── 6. EXPORT ─────────────────────────────────────────────────────────
                 ogr2ogr → output.gpkg
                   layer "edge" ← export_edge
                   layer "node" ← export_node
```

All state lives in PostgreSQL. Each step reads from and writes to named tables. The `on_existing` setting controls what happens if a table already exists (`skip` / `delete` / `abort`).

---

## 4. Entry Point - `generate_index.py`

The single entry point for the entire pipeline. It is invoked as:

```bash
python generate_index.py <settings_file.yml> [--skip import optional network attributes index export] [--loglevel 1|2|3|4]
```

Inside Docker, the image `ENTRYPOINT` is `python ./generate_index.py`, so the settings file is passed as `command` in `docker-compose.yml`.

### What it does

1. **Parse CLI arguments** - reads the settings file path, optional `--skip` list, and log level.
2. **Load YAML settings** - parses the settings file with `yaml.safe_load`.
3. **Apply global settings** - sets `GlobalSettings.custom_srid` and `GlobalSettings.case_id` from the `global` section if provided. The `case_id` is sanitised to alphanumeric + underscore only.
4. **Build `DbSettings`** - reads database connection parameters, falling back to `DB_USERNAME` and `DB_PASSWORD` environment variables if not in the YAML.
5. **Validate required sections** - before running anything, checks that `import`, `export`, and `profiles` sections exist (unless the corresponding steps are skipped).
6. **Execute steps in order** - calls each step's factory function and `run_step()` method unless that step is in the `--skip` list.

### Step skipping

Any combination of steps can be skipped with `--skip`. This is useful for re-running only part of the pipeline (e.g. re-running index computation with a different profile without re-importing data):

```bash
python generate_index.py settings.yml --skip import optional network attributes
```

### Log levels

| Level | Constant | Description |
|---|---|---|
| 1 | `LOG_LEVEL_1_MAJOR_INFO` | Only high-level phase markers |
| 2 | `LOG_LEVEL_2_INFO` | Standard informational output (default) |
| 3 | `LOG_LEVEL_3_DETAILED` | Detailed step-by-step output |
| 4 | `LOG_LEVEL_4_DEBUG` | Full debug including SQL queries and bind params |

---

## 5. Settings System - `settings.py`

### `InputType` (Enum)

```python
class InputType(Enum):
    OSM = "OSM"
    GIP = "GIP"
```

Used by factory functions (`create_importer`, `create_network_step`, `create_attributes_step`) to dispatch to the correct implementation.

### `GlobalSettings`

A class of static/class-level variables shared across the entire run:

| Attribute | Default | Description |
|---|---|---|
| `data_directory` | `"data"` | Directory for all input/output files |
| `osm_download_prefix` | `"osm_download"` | Prefix for downloaded OSM files |
| `overpass_api_endpoints` | 5 endpoints | Tried in order on failure |
| `default_srid` | `32633` (UTM 33N) | Used if no custom SRID and no AOI centroid detection |
| `custom_srid` | `None` | Set from `global.target_srid` in settings file |
| `case_id` | `"default_net"` | Set from `global.case_id`; used in schema names and filenames |

`get_target_srid()` returns `custom_srid` if set, otherwise `default_srid`.

### `DbSettings` (dataclass)

Holds all database connection parameters: `host`, `port`, `dbname`, `username`, `password`, `on_existing`.

Created via `DbSettings.from_dict(settings_dict)` which reads from the YAML `database` section and falls back to `DB_USERNAME` / `DB_PASSWORD` environment variables if credentials are absent.

On construction, `__post_init__` creates a `DbEntitySettings` instance at `self.entities`.

### `DbEntitySettings`

Derives all database schema and table names from `case_id`. Uses Python properties with computed defaults and optional manual overrides:

| Property | Default value | Example |
|---|---|---|
| `data_schema` | `netascore_data` | Shared across all cases |
| `network_schema` | `netascore_case_<case_id>` | e.g. `netascore_case_salzburg` |
| `output_schema` | same as `network_schema` | |

This means each `case_id` gets its own PostgreSQL schema for network/index tables, while all cases share one data schema for imported enrichment data (DEM, noise, etc.).

---

## 6. Toolbox

### `helper.py` - Logging & Utilities

**Logging functions:**

| Function | Log level | Description |
|---|---|---|
| `majorInfo(msg)` | 1 | Major phase announcements |
| `info(msg)` | 2 | Standard progress info |
| `log(msg, level)` | any | General log with timestamp |
| `debugLog(msg)` | 4 | Debug-only output |
| `logBeginTask(s)` | 2 | Prints task header + starts timer |
| `logEndTask()` | - | Prints duration since `logBeginTask` |

All output includes elapsed time since process start (format: `H:MM:SS`). Task timing uses `time.perf_counter` for precision. An `atexit` handler prints total run time on exit.

**Validation helpers:**

- `require_keys(settings, keys, error_message)` - calls `sys.exit(1)` if any key is missing from the dict
- `has_keys(settings, keys)` - returns bool, logs missing keys at debug level
- `has_any_key(settings, keys)` - returns True if at least one key is present

**SQL-safe string utilities** (used when building dynamic SQL to prevent injection):

- `get_safe_name(value)` - strips all non-alphanumeric/underscore characters
- `get_safe_string(value)` - allows alphanumeric, `_`, `.`, `:`, space, `-`
- `str_to_numeric(value)` - extracts numeric portion from a string; returns `int` or `float`
- `str_is_numeric_only(value)` - returns True if string contains only digits/decimal/minus
- `is_numeric(value)` - True for `int` or `float` type

---

### `dbhelper.py` - Database Abstraction

`PostgresConnection` wraps `psycopg2` and provides a high-level interface used by every pipeline step.

**Connection:**

```python
db = PostgresConnection.from_settings_object(db_settings)
db.connect()       # lazy: skips if already connected
db.schema = schema # sets search_path
db.close()
```

Connection uses keepalives (`keepalives_idle=5`, `keepalives_interval=2`, `keepalives_count=2`) and a 3-second connect timeout.

**Two connection string formats** are maintained for compatibility with different tools:

- `connection_string` - URL format for `psycopg2` and `raster2pgsql`: `postgresql://user:pw@host:port/db`
- `connection_string_old` - key=value format for `ogr2ogr` and legacy tools: `dbname='...' host='...'...`

**Schema and extension management:**

```python
db.init_extensions_and_schema(schema)
# Creates postgis, postgis_raster, hstore extensions and the schema, sets search_path
```

**Table conflict handling:**

```python
db.handle_conflicting_output_tables(['table1', 'table2'])
# Returns True  → tables don't exist, or were deleted (on_existing=delete)
# Returns False → tables exist and on_existing=skip (caller should skip the step)
# Raises        → tables exist and on_existing=abort
```

This method is called at the start of every significant write operation, providing consistent conflict resolution across the whole pipeline.

**SQL execution:**

```python
db.execute_sql_from_file("function_name", "sql/functions")
# Reads sql/functions/function_name.sql and executes it

db.execute_template_sql_from_file("template_name", params)
# Reads sql/templates/template_name.sql.j2, renders with JinjaSql, executes
```

Templates use `{{ param | sqlsafe }}` for identifiers (schema names, table names) and `{{ param }}` for values. JinjaSql translates value parameters to `%(name)s` pyformat placeholders, which `psycopg2` binds safely.

**Other utilities:**

- `db.exists(entity, schema)` - checks via `to_regclass()`
- `db.use_if_exists(entity, schema)` - returns the identifier string if it exists, else `None`
- `db.column_exists(column, schema, table)` - checks `information_schema.columns`
- `db.vacuum(table)` - switches to autocommit mode (required for `VACUUM`), runs `VACUUM FULL ANALYZE`, resets
- `db.geom_reproject(table, geomType, srid)` - in-place `ST_Transform` via `ALTER TABLE`
- `db.query_one(sql, vars)` / `db.query_all(sql, vars)` - execute + fetch shorthands

---

## 7. Core Pipeline Steps

All step classes inherit from `DbStep`:

```python
class DbStep:
    db_settings: DbSettings

    def __init__(self, db_settings: DbSettings):
        self.db_settings = db_settings

    def run_step(self, settings: dict):
        raise NotImplementedError()
```

Each module exposes a factory function that selects the GIP or OSM implementation based on `import_type`.

---

### Step 1: Import - `core/import_step.py`

Ingests raw source data into PostgreSQL. This is the most complex step, with very different logic for the two data sources.

#### Module-level functions

**`create_csv(file_txt)`** - Converts a GIP `.txt` file to `.csv`. GIP files use a custom line-prefix format:
- Lines starting with `tbl;` open a new output file
- Lines with `atr;` become the CSV header
- Lines with `rec;` become data rows (with empty-string cleaning)

**`create_sql(file_txt)`** - Parses a GIP `.txt` file and generates a `CREATE TABLE` SQL statement. Maps GIP type strings to PostgreSQL types:

| GIP type | PostgreSQL type |
|---|---|
| `string` | `varchar` |
| `string(N)` | `varchar(N)` |
| `decimal(p,s)` | `numeric(p,s)` |
| `decimal(p)` ≤4 | `smallint` |
| `decimal(p)` ≤10 | `integer` |
| `decimal(p)` ≤18 | `bigint` |

Reserved SQL keyword `offset` is renamed to `offset_`.

**`import_csv(connection_string, path, schema, table)`** - Runs `psql \copy` for bulk CSV loading (semicolon-delimited, UTF-8, with header).

**`import_geopackage(connection_string, path, schema, table, ...)`** - Uses `ogr2ogr` to import one or more layers from a GeoPackage into PostGIS. Supports:
- `fid` - FID column name
- `target_srid` - reprojects on import
- `layers` - which layers to import (defaults to all)
- `attributes` - column selection
- `geometry_types` - filters features by geometry type using `ST_GeometryType()` in a `-where` clause

**`import_osm(connection_string, path, path_style, schema, prefix)`** - Runs `osm2pgsql` with `--slim --hstore --latlong` and the provided style file. Creates `osm_point`, `osm_line`, `osm_polygon` tables.

#### `GipImporter`

Imports the GIP OGD dataset from a ZIP archive.

**Input tables loaded:**

| Filename | DB table | Primary key |
|---|---|---|
| `BikeHike.txt` | `gip_bikehike` | `use_id` |
| `Link.txt` | `gip_link` | `link_id` |
| `LinkCoordinate.txt` | `gip_linkcoordinate` | `link_id, count` |
| `LinkUse.txt` | `gip_linkuse` | `use_id` |
| `Link2ReferenceObject.txt` | `gip_link2referenceobject` | `idseq` |
| `Node.txt` | `gip_node` | `node_id` |
| `ReferenceObject.txt` | `gip_referenceobject` | `refobj_id` |

For each file: extracts from ZIP (if not already extracted) → converts `.txt` to `.csv` and `.sql` → drops existing table → executes `CREATE TABLE` SQL → bulk-loads CSV → adds primary key.

#### `OsmImporter`

Imports OpenStreetMap data, either from disk or downloaded live via the Overpass API.

**AOI-based download flow (`place_name`):**

1. Creates an `aoi` table if absent (columns: `id`, `name`, `geom`, `srid`)
2. Queries Overpass API for an administrative boundary polygon matching `place_name`
3. Handles multiple results - uses first, or prompts interactively if `interactive: True`
4. Stores AOI geometry in the `aoi` table
5. Computes the target SRID from the AOI centroid's UTM zone (via `determine_utmzone` SQL function) or uses `global.target_srid`
6. Expands the AOI bounding box by `buffer` metres (default 500)
7. Calls `_load_osm_from_bbox` with the expanded bounding box

**Bounding box download flow (`bbox` or derived from `place_name`):**

Queries Overpass API with a `nwr(__bbox__)` query (timeout 900s, max size 1GB). Tries each endpoint in `GlobalSettings.overpass_api_endpoints` sequentially on failure.

**After import** (`osm2pgsql`), derives ancillary spatial datasets from the raw OSM tables:

| Dataset | Source | Geometry | Filter |
|---|---|---|---|
| `building` | `osm_polygon` | Polygon | `building IS NOT NULL` |
| `crossing` | `osm_point/line/polygon` | Mixed | `highway = 'crossing'` |
| `facility` | `osm_point/polygon` | Mixed | Extensive amenity/tourism tag list |
| `greenness` | `osm_polygon` | Polygon | `landuse`, `leisure`, `natural` green types |
| `water` | `osm_line/polygon` | Mixed | `waterway IS NOT NULL OR natural = 'water'`, no tunnels |

All derived tables are reprojected to `target_srid` and spatially indexed (`USING gist`).

---

### Step 2: Optional Imports - `core/optional_step.py`

Imports supplementary datasets that enrich network attributes but are not required for a basic run.

**`import_raster(connection_string, path, schema, table, input_srid)`** - Pipes `raster2pgsql` into `psql`. Imports without reprojection; the network geometry is reprojected to the DEM's CRS temporarily during attribute calculation, not the other way round.

| Importer class | Dataset | Format | Notes |
|---|---|---|---|
| `DemImporter` | Digital Elevation Model | GeoTIFF | Elevation for gradient computation |
| `NoiseImporter` | Noise exposure | GeoPackage (Polygon) | Noise level in dB |
| `BuildingImporter` | Building footprints | GeoPackage (Polygon) | |
| `CrossingImporter` | Crossings | GeoPackage (Point, Line) | |
| `FacilityImporter` | Points of interest | GeoPackage (Point, Polygon) | |
| `GreennessImporter` | Green spaces | GeoPackage (Polygon) | |
| `WaterImporter` | Water bodies | GeoPackage (Line, Polygon) | |

`run_optional_importers(db_settings, settings_dict)` iterates over the `optional` section of the settings file and runs each importer. The `osm` optional type delegates to `OsmImporter` from `import_step`.

---

### Step 3: Network Construction - `core/network_step.py`

Transforms raw imported data into a topologically correct, routable network graph: `network_edge` and `network_node` tables.

#### `GipNetworkStep` → `gip_network.sql.j2`

The GIP data is already a well-formed graph. The template:

1. Reconstructs linestring geometry by ordering coordinates from `gip_linkcoordinate` between `gip_node` start/end points
2. Filters to edges with at least car, bicycle, or pedestrian access (bitmask check on `access_tow`/`access_bkw`) and excludes parking garages (`formofway <> 7`)
3. Joins `gip_linkuse` and `gip_bikehike` for cycling-specific attributes
4. Creates `network_edge` with geometry, speed/access/lane/width attributes
5. Creates `network_node` filtered to only nodes referenced by `network_edge`
6. Drops all temporary tables

#### `OsmNetworkStep` → `osm_network.sql.j2`

OSM data requires extensive topology work since OSM ways are not pre-split at intersections.

1. Selects all road/path/rail/aerialway features from `osm_line`, handling bridge and tunnel edge cases
2. Calculates start/end points for each way
3. Finds all intersection points between ways (respecting layer, bridge, tunnel context to avoid false splits)
4. Splits ways at intersections to create proper edge segments
5. Assigns unique `edge_id` (sequence-based)
6. Runs `osm_delete_dangling_edges` function to clean up dead-end stubs
7. Creates `network_node` from unique start/end points of all edges

Both steps create:
- `network_edge(edge_id PK, from_node, to_node, geom, length, ...)` with spatial index
- `network_node(node_id PK, geom, elevation)` with spatial index

---

### Step 4: Attribute Calculation - `core/attributes_step.py`

Computes all transport-relevant indicators per edge/node and writes to `network_edge_attributes`, `network_node_attributes`, and `network_edge_export`.

#### Common optional table handling

Both GIP and OSM attribute steps accept the same set of optional enrichment tables. Each is passed via `db.use_if_exists()`, which returns the table identifier if present or `None` if absent. Attributes derived from absent tables are simply not computed.

| Parameter | Table | Derived attributes |
|---|---|---|
| `table_dem` | `dem` | Node elevation, edge gradient |
| `table_noise` | `noise` | Noise level per edge |
| `table_building` | `building` | Building density within 30m buffer |
| `table_crossing` | `crossing` | Number of crossings within 10m buffer |
| `table_facility` | `facility` | Number of POIs within 30m buffer |
| `table_greenness` | `greenness` | Green space fraction within 30m buffer |
| `table_water` | `water` | Water body presence within 30m buffer |

#### `GipAttributesStep` → `gip_attributes.sql.j2`

Registers three SQL functions first:
- `gip_calculate_bicycle_infrastructure` - maps GIP `bikeenvironment`/`bikefeature` flags to `bicycle_infrastructure` categories
- `gip_calculate_pedestrian_infrastructure` - maps GIP attributes to `pedestrian_infrastructure` categories
- `gip_calculate_road_category` - maps GIP `funcroadclass`/`streetcat` to road category strings

**Note:** If a DEM is provided, a warning is emitted and it is ignored - GIP contains its own elevation data.

#### `OsmAttributesStep` → `osm_attributes.sql.j2`

Registers three SQL functions:
- `osm_calculate_access_bicycle` - interprets `bicycle`, `foot`, `access`, `cycleway`, `highway` tags + directionality
- `osm_calculate_access_car` - interprets `motor_vehicle`, `access`, `oneway` tags + roundabout logic
- `osm_calculate_access_pedestrian` - interprets `foot`, `access`, `highway` tags + directionality

These functions return boolean `_ft`/`_tf` access flags. The OSM access logic is significantly more complex than GIP due to OSM's tag variety.

#### Output tables

After attributes are calculated, a `network_edge_export` view/table is created joining:
- `network_edge` (geometry, topology)
- `network_edge_attributes` (all computed indicators)

This is what the index step reads.

---

### Step 5: Index Generation - `core/index_step.py`

The most algorithmically complex step. Translates YAML mode profile definitions into PostgreSQL PL/pgSQL functions at runtime, then uses those functions to compute composite routing indices.

#### Profile loading

```python
profiles = load_profiles(base_path, profile_definitions)
```

`ModeProfile` reads a YAML profile file and parses:
- `profile_name` - used in output column names (e.g. `index_bike_ft`)
- `access_car/bike/walk` - filter flags from `filter_access_*` keys; if none set, all modes enabled
- `profile` - the full parsed YAML dict (weights, overrides, indicator_mapping)

#### YAML → SQL translation

The profile YAML is translated into SQL `CASE...END` expressions that become the body of the `calculate_index()` PL/pgSQL function.

**`_build_sql_indicator_mapping_internal_(indicator_yml)`**

Recursively parses an indicator mapping block. Produces a SQL `CASE...END` expression:

```sql
CASE
    WHEN indicator_name = 'value' THEN score
    WHEN indicator_name IN ('v1', 'v2') THEN score
    WHEN indicator_name IS NULL THEN score
    WHEN indicator_name >= 50 THEN score
    ...
    ELSE default_score
END
```

Supports:
- **`mapping`** - exact equality checks. String, numeric, boolean, and list values `{v1,v2}` are all handled. Numeric lists get `IN (n1, n2)`; string lists get `IN ('s1', 's2')`.
- **`classes`** - range-based. Keys use operator prefixes (`g`=`>`, `ge`=`>=`, `l`=`<`, `le`=`<=`, `e`=`=`, `ne`=`<>`) prepended to the threshold value (e.g. `ge50: 0.6` → `WHEN indicator >= 50 THEN 0.6`).
- **Nested dicts** - the assigned value can itself be another indicator block, producing a nested `CASE...END`.
- **`NULL` key** - generates `WHEN indicator IS NULL THEN value`.
- **`_default_` key** - generates `ELSE value`.

All string values are passed through `get_safe_string()` to prevent SQL injection.

**`_build_sql_indicator_mapping(indicator_yml)`**

Wraps the `CASE...END` in the full indicator contribution block:

```sql
IF indicator_name IS NOT NULL AND indicator_name_weight IS NOT NULL THEN
    indicator := <CASE...END>;
    weight := indicator_name_weight / weights_sum;
    index := index + indicator * weight;
    indicator_weights := array_append(indicator_weights, ('name', indicator * weight)::indicator_weight);
END IF;
```

This computes the normalised weighted contribution of each indicator to the composite index.

**`_build_sql_overrides(overrides_yml)`**

Generates override blocks that can either:
- Set specific indicator **weights** (type `weight`) - useful for penalising combinations like steep + unpaved
- Set the **composite index** directly (type `index`) and return early - useful for fixed-score edge cases

Example: for the walk profile, when `pedestrian_infrastructure = 'sidewalk'` AND `road_category IN ('primary', 'secondary')`, the index is fixed to `0.2` without running the weighted sum.

#### Function registration and execution

For each profile:

1. The compiled SQL fragments are inserted into `calculate_index.sql.j2` template:
   ```sql
   CREATE OR REPLACE FUNCTION calculate_index(
       IN bicycle_infrastructure varchar, IN bicycle_infrastructure_weight numeric,
       ... 18 indicator+weight pairs ...
       OUT index numeric,
       OUT index_robustness numeric,
       OUT index_explanation json
   ) AS $$ ... $$
   ```
   The function receives all indicators and weights as parameters, runs the override logic first (allowing early return), then sums the weighted indicators.

2. The `index.sql.j2` template is executed per profile, using `LATERAL calculate_index(...)` to compute both `_ft` and `_tf` index values for every edge in a single pass.

3. Columns added to `network_edge_index`:
   - `index_<profile>_ft` / `_tf` - the composite index value (0–1)
   - `index_<profile>_ft_robustness` / `_tf_robustness` - fraction of non-null indicator weights (data completeness indicator)
   - `index_<profile>_ft_explanation` / `_tf_explanation` - JSON array of per-indicator weighted contributions (only if `compute_explanation: True`)

4. After all profiles are computed, `export.sql.j2` joins `network_edge_export`, `network_edge_attributes`, and `network_edge_index` into `export_edge`, and joins `network_node` with `network_node_attributes` into `export_node`.

5. A final check queries `pg_stats` for any fully-null columns in `export_edge` and emits a warning if found (indicates data gaps).

---

### Step 6: Export - `core/export_step.py`

Exports the final network tables to a GeoPackage file using `ogr2ogr`.

**`export_geopackage(connection_string, path, schema, table, layer, fid, geometry_type, update)`**

Runs:
```bash
ogr2ogr -f "GPKG" output.gpkg PG:"..." -lco FID=edge_id -lco GEOMETRY_NAME=geom \
        -nln edge -nlt LINESTRING -progress -sql "SELECT * FROM schema.table"
```

#### `GeopackageExporter.run_step(settings)`

1. Resolves output filename: replaces `<case_id>` and `<srid>` placeholders
2. Deletes the existing output file if it exists
3. Exports `export_edge` as the `edge` layer (geometry: `LINESTRING`, FID: `edge_id`)
4. Exports `export_node` as the `node` layer (geometry: `POINT`, FID: `node_id`, `-update` flag to append to the same file)

---

## 8. SQL Layer

### Templates (Jinja2)

All templates use JinjaSql via `dbhelper.execute_template_sql_from_file()`. Parameters passed as Python dicts are rendered into the template, with two kinds of substitution:

- `{{ param | sqlsafe }}` - unsafe: value is inserted directly into SQL string. Used for schema names, table names, SQL fragments (indicator mapping SQL). Only safe because all such values have been sanitised beforehand in Python.
- `{{ param }}` - safe: rendered to a `%(param)s` pyformat placeholder, bound by psycopg2. Used for scalar values.

Templates are stored in `sql/templates/` with `.sql.j2` extension.

**`VACUUM FULL ANALYZE` workaround:** The `dbhelper` replaces `VACUUM FULL ANALYZE` with `ANALYZE` in template SQL strings before execution. This is because `VACUUM` cannot run inside a transaction, but the template execution temporarily enables autocommit mode to handle this. The substitution is documented as a temporary workaround.

### Functions

Plain SQL files in `sql/functions/` define reusable PostgreSQL functions. They are executed once at the beginning of the attributes step via `execute_sql_from_file()`.

**GIP functions:**
- `gip_calculate_road_category` - maps `funcroadclass`, `streetcat`, `formofway` to road category string
- `gip_calculate_bicycle_infrastructure` - maps `bikeenvironment`, `bikefeature*` columns to infrastructure category
- `gip_calculate_pedestrian_infrastructure` - maps GIP attributes to pedestrian infrastructure category

**OSM functions:**
- `osm_calculate_access_car` - complex multi-tag access logic for motor vehicles, respecting directionality and one-way rules
- `osm_calculate_access_bicycle` - access logic for cyclists including `cycleway:*` tags and contraflow rules
- `osm_calculate_access_pedestrian` - access logic for pedestrians
- `osm_delete_dangling_edges` - removes topological dead-end artifacts from OSM network construction

**Shared functions:**
- `determine_utmzone` - calculates appropriate UTM SRID from a geometry centroid's longitude
- `calculate_index.sql.j2` - the dynamic PL/pgSQL function template. Receives the indicator mapping SQL and override SQL as Jinja parameters, producing a complete function definition that is re-registered for each mode profile.

---

## 9. Mode Profiles

Mode profiles are YAML files that fully define the scoring logic for one transport mode. They are referenced from the settings file and translated to SQL at runtime. No code changes are needed to add or modify a profile.

### Profile structure

```yaml
version: 1.1

weights:          # Relative importance of each indicator; NULL to exclude
  road_category: 0.3
  gradient: 0.1
  noise: NULL     # excluded from this profile

overrides:        # Rules that bypass the normal weighted sum
  - description: ...
    indicator: primary_indicator
    output:
      type: index  # or: weight
      for: [ind1, ind2]  # only for type: weight
    mapping:
      "value": score_or_nested_mapping

indicator_mapping:  # Maps raw attribute values to [0,1] scores
  - indicator: road_category
    mapping:
      "primary": 0
      "residential": 0.8
  - indicator: gradient
    classes:
      ge6: 0
      ge3: 0.4
      ge0: 0.9
```

### Weights

Weights do not need to sum to 1 - the index computation normalises by `weights_sum` (the sum of non-null weights for indicators that have non-null data for a given edge).

`index_robustness` = `weights_sum / weights_total` represents the fraction of the configured weight that had actual data, providing a data quality/completeness indicator per edge.

### Default bike profile (`profile_bike.yml`)

Key weights: `road_category` (0.3) > `bicycle_infrastructure` (0.2) > `designated_route`, `max_speed`, `parking`, `pavement`, `gradient` (0.1 each). Environmental indicators (`greenness`, `water`, `noise`, etc.) are available but set to `NULL` by default for a lean, infrastructure-focused assessment.

One override: steep gradient (class ±3 or ±4) combined with loose/rough surface (`gravel`, `soft`, `cobble`) boosts both `pavement` and `gradient` weights to `1.6`, penalising these combinations more strongly.

### Default walk profile (`profile_walk.yml`)

Key weights: `pedestrian_infrastructure` (0.4), `water` (0.4), `road_category` / `gradient` / `facilities` / `greenness` / `noise` (0.3 each), `crossings` (0.2), `buildings` (0.1).

One override: `pedestrian_infrastructure = 'sidewalk'` on `primary` or `secondary` roads fixes the index to `0.2`, preventing sidewalks on fast roads from scoring normally via the weighted sum.

The `crossings` indicator uses a nested mapping: if `crossings = 0` (no crossings nearby), the score is determined by `road_category` - no crossings on a `primary` or `secondary` road scores `0`; on a residential road scores `0.5`. This captures the danger of crossing a busy road with no dedicated crossing point.

---

## 10. Configuration Reference

### Full settings file schema

```yaml
version: 1.2            # Required

global:                  # Optional
  case_id: my_area       # Alphanumeric + underscore only
  target_srid: 32633     # UTM EPSG code; required for bbox import or OSM file import

database:                # Optional; defaults to internal Docker DB
  host: netascore-db     # Use 'gateway.docker.internal' for host DB from Docker
  port: 5432
  dbname: postgres
  username: postgres     # Or set DB_USERNAME env var
  password: postgres     # Or set DB_PASSWORD env var
  on_existing: abort     # skip | delete | abort

import:                  # Required (unless skipped)
  type: osm              # osm | gip
  # --- OSM options ---
  filename: area.osm.pbf       # Local file (alternative to API download)
  place_name: Salzburg         # Overpass AOI query by name
  bbox: 47.79,13.01,...        # Overpass bounding box (requires target_srid)
  on_existing: delete          # For downloaded OSM file conflict
  buffer: 1000                 # AOI expansion in metres
  admin_level: 8               # Overpass admin_level filter
  zip_code: 5020               # Overpass postal code filter
  interactive: False           # Prompt if multiple AOI matches
  include_rail: False          # Include railway geometry
  include_aerialway: False     # Include aerialway geometry
  filename_style: default.style # Custom osm2pgsql style file
  # --- GIP options ---
  filename_A: A_routingexport_ogd_split.zip

optional:                # Optional
  dem:
    filename: dem.tif
    srid: 31287
  noise:
    filename: noise.gpkg
  osm:                   # Only for GIP import; derives building/crossing/facility/greenness/water
    filename: area.osm.pbf
  building:
    filename: building.gpkg
  crossing:
    filename: crossing.gpkg
  facility:
    filename: facility.gpkg
  greenness:
    filename: greenness.gpkg
  water:
    filename: water.gpkg

index:                   # Optional
  compute_explanation: False  # True adds JSON explanation column to output

profiles:                # Required (unless skipped)
  - profile_name: bike
    filename: profile_bike.yml
    filter_access_bike: True   # filter_access_car | filter_access_bike | filter_access_walk

export:                  # Required (unless skipped)
  type: geopackage
  filename: output_<case_id>.gpkg  # <case_id> and <srid> are replaced at runtime
```

### CLI reference

```bash
python generate_index.py <settings.yml> [options]

Options:
  --skip {import,optional,network,attributes,index,export} [...]
                        Skip one or more pipeline steps
  --loglevel {1,2,3,4}  1=major, 2=info (default), 3=detailed, 4=debug
```

---

## 11. Deployment

### Docker (recommended)

The pre-built image `plusmobilitylab/netascore:latest` packages all dependencies.

**Quickstart (OSM query for Salzburg):**

```bash
# Download docker-compose.yml from examples/ to an empty directory
docker compose run netascore
# Output: data/netascore_salzburg.gpkg
```

**Custom area:**

```bash
mkdir data
cp examples/settings_osm_query.yml data/my_settings.yml
cp examples/profile_bike.yml examples/profile_walk.yml data/
# Edit data/my_settings.yml: change case_id and place_name
docker compose run netascore data/my_settings.yml
```

**Docker Compose services:**

| Service | Image | Port | Notes |
|---|---|---|---|
| `netascore` | `plusmobilitylab/netascore:latest` | - | Mounts `./data` into container |
| `netascore-db` | `postgis/postgis:13-3.2` | 5433:5432 | Health check: `pg_isready` |

The `netascore` service waits for `netascore-db` to pass its health check (up to 120 × 10s retries) before starting.

### Database topology options

| Scenario | `database.host` |
|---|---|
| Both in Docker (default) | `netascore-db` |
| NetAScore in Docker, DB on host | `gateway.docker.internal` |
| Both on host (no Docker) | `localhost` |

### Environment variables

```bash
DB_USERNAME=myuser DB_PASSWORD=mypassword docker compose run netascore data/settings.yml
```

Credentials set as environment variables take precedence over YAML values.

### Build from source

```bash
docker compose build          # builds using Dockerfile
docker compose run netascore data/settings.yml
```

The Dockerfile uses `python:3.8.17-bullseye` and installs `gdal-bin`, `postgresql-client-13`, `osm2pgsql` via `apt`, and Python dependencies via `pip`.

### Performance notes

- On macOS/Windows, Docker volume mounts are 3–5× slower than Linux. For large datasets, copy files into a Docker volume instead:
  ```bash
  docker volume create netascore-storage
  docker create --name netascore-pipe -v netascore-storage:/usr/src/netascore/data netascore data/settings.yml
  docker cp data/. netascore-pipe:/usr/src/netascore/data
  docker start netascore-pipe
  ```
- For memory errors on large datasets, add `shm_size: 2gb` to the `netascore-db` service in `docker-compose.yml`.

---

## 12. Output Data Model

The final output GeoPackage contains two layers.

### `edge` layer - `export_edge`

A join of `network_edge_export` + `network_edge_attributes` + `network_edge_index`.

**Topology columns:**

| Column | Type | Description |
|---|---|---|
| `edge_id` | integer PK | Unique edge identifier |
| `from_node` | integer | Start node |
| `to_node` | integer | End node |
| `geom` | LineString | Edge geometry in target SRID |
| `length` | numeric | Length in CRS units (metres) |

**Access flags** (boolean, directional `_ft`/`_tf`):

`access_car_ft/tf`, `access_bicycle_ft/tf`, `access_pedestrian_ft/tf`

**Structural attributes** (boolean):

`bridge`, `tunnel`

**Directional indicators** (`_ft`/`_tf` variants where applicable):

| Column | Type | Values |
|---|---|---|
| `bicycle_infrastructure_*` | varchar | `bicycle_way`, `mixed_way`, `bicycle_road`, `cyclestreet`, `bicycle_lane`, `bus_lane`, `shared_lane`, `undefined`, `no` |
| `pedestrian_infrastructure_*` | varchar | `pedestrian_area`, `pedestrian_way`, `mixed_way`, `stairs`, `sidewalk`, `no` |
| `designated_route_*` | varchar | `international`, `national`, `regional`, `local`, `unknown`, `no` |
| `road_category` | varchar | `primary`, `secondary`, `residential`, `service`, `calmed`, `no_mit`, `path` |
| `max_speed_*` | numeric | 0–130 km/h |
| `max_speed_greatest` | numeric | Max of both directions |
| `parking_*` | varchar | `yes`, `no` |
| `pavement` | varchar | `asphalt`, `gravel`, `cobble`, `soft` |
| `width` | numeric | Metres |
| `gradient_*` | integer | −4 to +4 (see class table) |
| `number_lanes_*` | integer | Lane count |

**Buffer-based indicators** (non-directional):

| Column | Type | Description |
|---|---|---|
| `facilities` | integer | POI count within 30m buffer |
| `crossings` | integer | Crossing count within 10m buffer |
| `buildings` | numeric | Building area % within 30m buffer (0–100) |
| `greenness` | numeric | Green area % within 30m buffer (0–100) |
| `water` | boolean | Water presence within 30m buffer |
| `noise` | numeric | Noise level in dB |

**Index columns** (per profile, e.g. `bike`):

| Column | Type | Description |
|---|---|---|
| `index_bike_ft` | numeric | Bikeability forward direction (0–1) |
| `index_bike_tf` | numeric | Bikeability reverse direction (0–1) |
| `index_bike_ft_robustness` | numeric | Data completeness (0–1) |
| `index_bike_tf_robustness` | numeric | Data completeness (0–1) |
| `index_bike_ft_explanation` | json | Per-indicator contributions (if enabled) |
| `index_bike_tf_explanation` | json | Per-indicator contributions (if enabled) |

The same columns are repeated for each additional profile (`walk`, or any custom profile name).

### `node` layer - `export_node`

| Column | Type | Description |
|---|---|---|
| `node_id` | integer PK | Unique node identifier |
| `geom` | Point | Node geometry in target SRID |
| `elevation` | numeric | Elevation in metres (if DEM provided) |

---

## 13. Dependencies

### Python libraries (`requirements.txt`)

| Library | Usage |
|---|---|
| `psycopg2` | PostgreSQL database adapter |
| `PyYAML` | Settings and profile file parsing |
| `JinjaSql` + `Jinja2<3.1.0` | SQL template rendering |
| `gdal==3.2.2` (osgeo) | GeoPackage layer inspection (`ogr.Open`) |
| `requests` | Overpass API HTTP calls |
| `osm2geojson` | AOI XML response → GeoJSON conversion |
| `pandas` | (available but not directly used in pipeline) |
| `Flask==2.0.3` | (available; not used in core pipeline) |
| `igraph` | (available; not used in core pipeline) |

### System tools

| Tool | Used in | Purpose |
|---|---|---|
| `psql` | `import_step` | `\copy` CSV bulk loading |
| `ogr2ogr` (GDAL) | `import_step`, `optional_step`, `export_step` | GeoPackage import/export |
| `osm2pgsql` | `import_step` | OSM `.pbf`/`.xml` → PostGIS |
| `raster2pgsql` | `optional_step` | GeoTIFF → PostGIS raster |
| `PostgreSQL ≥13` with `PostGIS`, `postgis_raster`, `hstore` | All steps | Database backend |
