# Add Tree Shade Cover (`shade_coverage`) to NetAScore

This document captures every code change required to add `shade_coverage` as a new optional
indicator in NetAScore, using the Meta/WRI 1m Global Canopy Height raster as input.

---

## Overview

| Item | Detail |
|---|---|
| Data source | Meta/WRI 1m Global Canopy Height Maps (Tolan et al. 2024) |
| GEE collection | `projects/sat-io/open-datasets/facebook/meta-canopy-height` |
| Input format | Single-band GeoTIFF, clipped to city extent |
| Pixel values | 0–31 (metres); 0 = no canopy (valid data, not NoData) |
| Output column | `shade_coverage` - float in [0, 1] on the edge table |
| Walk weight (MAHP) | 0.0999 (rank 5 of 10) |
| Bike weight (MAHP) | 0.0892 (rank 7 of 10) |

### Scoring curve (height-weighted, concave)

Calibrated to Indian urban street trees. Both density and height drive the score -
all pixels in the 10m buffer contribute to the average, including zeros.

| Canopy height | Score | Species context |
|---|---|---|
| 0 – 3 m | 0.00 | No canopy / shrubs / saplings - no meaningful shade |
| 3 – 8 m | 0.30 → 0.70 | Young street trees |
| 8 – 12 m | 0.70 → 0.90 | Mature neem, peepal |
| 12 – 18 m | 0.90 → 1.00 | Excellent canopy |
| ≥ 18 m | 1.00 | Rain tree / banyan - capped |

> Anything below 3m scores 0 - the threshold reflects the minimum canopy height
> needed to provide meaningful overhead shade during hot seasons.

---

## Files to create

### 1. `sql/templates/attribute_shade_coverage.sql.j2` - new file

```sql
-- =============================================================================
-- Attribute: shade_coverage
-- Source:    Meta/WRI 1m Global Canopy Height (Tolan et al. 2024)
-- Logic:     For each edge, buffer 10m → dump pixels via ST_PixelAsPoints →
--            apply height-weighted scoring curve → AVG across all pixels
--            Result is in [0, 1].
--
-- Scoring curve (concave - calibrated to Indian urban street trees):
--   0 ≤ h < 3m  → 0.00  (no canopy / shrubs / saplings - no meaningful shade)
--   3 ≤ h < 8m  → 0.30 + (h-3) × 0.08   [0.30 – 0.70]  (young street trees)
--   8 ≤ h < 12m → 0.70 + (h-8) × 0.05   [0.70 – 0.90]  (mature: neem, peepal)
--   12 ≤ h < 18m→ 0.90 + (h-12) × 0.0167 [0.90 – 1.00] (excellent canopy)
--   ≥ 18m       → 1.00  (rain tree / banyan - full shade, capped)
--
-- Note: pixel value 0 = no canopy (valid data, not NoData).
--       All pixels in the buffer contribute to the average (denominator
--       includes zeros), so density and height both drive the final score.
-- =============================================================================

ALTER TABLE {{ edge_table }}
    ADD COLUMN IF NOT EXISTS shade_coverage double precision;

UPDATE {{ edge_table }} e
SET shade_coverage = sub.score
FROM (
    SELECT
        e.edge_id,
        AVG(
            CASE
                WHEN p.val < 3              THEN 0.0
                WHEN p.val < 8              THEN 0.30 + (p.val - 3)  * (0.40 / 5.0)
                WHEN p.val < 12             THEN 0.70 + (p.val - 8)  * (0.20 / 4.0)
                WHEN p.val < 18             THEN 0.90 + (p.val - 12) * (0.10 / 6.0)
                ELSE                             1.0
            END
        ) AS score
    FROM {{ edge_table }} e
    JOIN {{ schema }}.shade_coverage_raster r
      ON ST_Intersects(r.rast, ST_Buffer(e.geom, 10))
    -- ST_PixelAsPoints dumps every pixel centroid + value within the clipped raster
    CROSS JOIN LATERAL (
        SELECT (ST_PixelAsPoints(
                    ST_Clip(r.rast, 1, ST_Buffer(e.geom, 10), 0, TRUE)
                )).*
    ) AS pixels(geom, val, x, y)
    -- keep only pixels whose centroid falls inside the buffer
    -- (ST_Clip can include partial edge tiles; this trims them cleanly)
    WHERE ST_Within(pixels.geom, ST_Buffer(e.geom, 10))
    GROUP BY e.edge_id
) sub
WHERE e.edge_id = sub.edge_id;

-- Edges with no raster coverage get 0 (no canopy data → assume no shade)
UPDATE {{ edge_table }}
SET shade_coverage = 0
WHERE shade_coverage IS NULL;
```

### 2. `examples/profile_walk_india.yml` - new file

```yaml
# =============================================================================
# profile_walk_india.yml
# India-adapted walkability profile - MAHP-calibrated weights
# Survey: KoboToolbox MAHP instrument, urban planning experts, Indian cities
# Reference: SOW_CONTEXT.md - Phase 1 weight calibration
# =============================================================================

name: walk_india
mode: walk

indicators:

  - name: pedestrian_infrastructure
    weight: 0.1562
    data_col: pedestrian_infrastructure
    value_mappings:
      - { from: 0, to: 0.0 }   # no infrastructure
      - { from: 1, to: 0.5 }   # partial / low-quality
      - { from: 2, to: 1.0 }   # dedicated footpath present

  - name: crossings
    weight: 0.1303
    data_col: crossings
    value_mappings:
      - { from: 0, to: 0.0 }
      - { from: 1, to: 0.6 }
      - { from: 2, to: 1.0 }

  - name: width
    weight: 0.1171
    data_col: width
    value_mappings:
      - { from: 0.0, to: 0.0 }
      - { from: 1.5, to: 0.5 }
      - { from: 3.0, to: 1.0 }

  - name: max_speed
    weight: 0.1137
    data_col: max_speed
    # Lower speed = better for pedestrians → inverted mapping
    value_mappings:
      - { from: 0,   to: 1.0 }
      - { from: 30,  to: 0.8 }
      - { from: 50,  to: 0.5 }
      - { from: 70,  to: 0.2 }
      - { from: 100, to: 0.0 }

  - name: shade_coverage
    weight: 0.0999
    data_col: shade_coverage
    # shade_coverage is 0–1 (height-weighted average over 10m buffer)
    # already scaled - pass through with linear identity mapping
    value_mappings:
      - { from: 0.0, to: 0.0 }
      - { from: 1.0, to: 1.0 }

  - name: road_category
    weight: 0.0898
    data_col: road_category
    value_mappings:
      - { from: 1, to: 0.0 }   # motorway / trunk
      - { from: 2, to: 0.2 }   # primary
      - { from: 3, to: 0.5 }   # secondary
      - { from: 4, to: 0.8 }   # tertiary / residential
      - { from: 5, to: 1.0 }   # living street / path

  - name: facilities
    weight: 0.0889
    data_col: facilities
    value_mappings:
      - { from: 0, to: 0.0 }
      - { from: 1, to: 0.6 }
      - { from: 2, to: 1.0 }

  - name: greenness
    weight: 0.0886
    data_col: greenness
    value_mappings:
      - { from: 0.0, to: 0.0 }
      - { from: 1.0, to: 1.0 }

  - name: gradient
    weight: 0.0727
    data_col: gradient
    # Steeper = worse for walking → inverted
    value_mappings:
      - { from: 0,  to: 1.0 }
      - { from: 5,  to: 0.8 }
      - { from: 10, to: 0.4 }
      - { from: 15, to: 0.1 }
      - { from: 20, to: 0.0 }

  - name: water
    weight: 0.0428
    data_col: water
    value_mappings:
      - { from: 0, to: 0.0 }
      - { from: 1, to: 1.0 }
```

### 3. `examples/profile_bike_india.yml` - new file

```yaml
# =============================================================================
# profile_bike_india.yml
# India-adapted bikeability profile - MAHP-calibrated weights
# Survey: KoboToolbox MAHP instrument, urban planning experts, Indian cities
# Reference: SOW_CONTEXT.md - Phase 1 weight calibration
# =============================================================================

name: bike_india
mode: bike

indicators:

  - name: pavement
    weight: 0.1319
    data_col: pavement
    # Surface quality - primary barrier for cyclists in India
    value_mappings:
      - { from: 0, to: 0.0 }   # unpaved / very poor
      - { from: 1, to: 0.3 }   # poor / gravel
      - { from: 2, to: 0.6 }   # asphalt, worn
      - { from: 3, to: 1.0 }   # asphalt, good

  - name: road_category
    weight: 0.1238
    data_col: road_category
    value_mappings:
      - { from: 1, to: 0.0 }   # motorway / trunk
      - { from: 2, to: 0.2 }   # primary
      - { from: 3, to: 0.5 }   # secondary
      - { from: 4, to: 0.8 }   # tertiary / residential
      - { from: 5, to: 1.0 }   # living street / path / cycleway

  - name: max_speed
    weight: 0.1217
    data_col: max_speed
    # Lower speed = better for cycling
    value_mappings:
      - { from: 0,   to: 1.0 }
      - { from: 30,  to: 0.8 }
      - { from: 50,  to: 0.5 }
      - { from: 70,  to: 0.2 }
      - { from: 100, to: 0.0 }

  - name: width
    weight: 0.1173
    data_col: width
    value_mappings:
      - { from: 0.0, to: 0.0 }
      - { from: 3.0, to: 0.5 }
      - { from: 6.0, to: 1.0 }

  - name: bicycle_infrastructure
    weight: 0.1055
    data_col: bicycle_infrastructure
    value_mappings:
      - { from: 0, to: 0.0 }   # no infrastructure
      - { from: 1, to: 0.5 }   # shared / advisory lane
      - { from: 2, to: 1.0 }   # dedicated cycle track

  - name: gradient
    weight: 0.0904
    data_col: gradient
    value_mappings:
      - { from: 0,  to: 1.0 }
      - { from: 3,  to: 0.8 }
      - { from: 6,  to: 0.5 }
      - { from: 10, to: 0.2 }
      - { from: 15, to: 0.0 }

  - name: shade_coverage
    weight: 0.0892
    data_col: shade_coverage
    # shade_coverage is 0–1 (height-weighted average over 10m buffer)
    # already scaled - pass through with linear identity mapping
    value_mappings:
      - { from: 0.0, to: 0.0 }
      - { from: 1.0, to: 1.0 }

  - name: greenness
    weight: 0.0890
    data_col: greenness
    value_mappings:
      - { from: 0.0, to: 0.0 }
      - { from: 1.0, to: 1.0 }

  - name: facilities
    weight: 0.0794
    data_col: facilities
    value_mappings:
      - { from: 0, to: 0.0 }
      - { from: 1, to: 0.6 }
      - { from: 2, to: 1.0 }

  - name: water
    weight: 0.0519
    data_col: water
    value_mappings:
      - { from: 0, to: 0.0 }
      - { from: 1, to: 1.0 }
```

---

## Files to modify

### 4. `settings.py` - add field to `OptionalSettings` dataclass

Add `shade_coverage_path` alongside the existing optional path fields (`dem_path`,
`noise_path`, `buildings_path`, etc.):

```python
@dataclass
class OptionalSettings:
    # ... existing fields ...
    dem_path:            Optional[str] = None
    noise_path:          Optional[str] = None
    buildings_path:      Optional[str] = None
    crossings_path:      Optional[str] = None
    facilities_path:     Optional[str] = None
    greenness_path:      Optional[str] = None
    water_path:          Optional[str] = None

    # NEW - Meta/WRI 1m canopy height raster clipped to city extent
    # Pixel values: 0 (no canopy) to 31 (metres). Single band GeoTIFF.
    # Source: https://gee-community-catalog.org/projects/meta_trees/
    # Citation: Tolan et al. (2024), Remote Sensing of Environment, 300, 113888.
    shade_coverage_path: Optional[str] = None
```

### 5. `core/optional_step.py` - add `ShadeCoverageStep` class

Add the class and its factory function alongside the existing optional layer classes
(e.g. `DemStep`). The import pattern is identical to DEM:

```python
class ShadeCoverageStep(DbStep):
    """
    Imports a Meta/WRI 1m canopy height GeoTIFF into PostGIS as a raster table.

    The raster is expected to have:
      - Single band, pixel values 0–31 (metres)
      - 0 = no canopy (valid data, not NoData)
      - Projected or geographic CRS matching the case SRID

    The imported table is: <data_schema>.shade_coverage_raster
    It is later consumed by attributes_step to derive the shade_coverage column.
    """

    def run_step(self, settings: GlobalSettings):
        path = settings.optional.shade_coverage_path
        if not path:
            log(2, "ShadeCoverageStep: no path configured, skipping.")
            return

        log(1, "Importing shade coverage raster (Meta/WRI canopy height)...")
        logBeginTask()

        schema = settings.database.data_schema          # netascore_data
        table  = "shade_coverage_raster"
        srid   = settings.global_settings.srid

        # Build raster2pgsql command - same flags used for DEM import
        cmd = [
            "raster2pgsql",
            "-s", str(srid),   # source SRID
            "-I",              # create spatial index
            "-C",              # apply raster constraints
            "-M",              # vacuum analyse after load
            "-t", "100x100",   # tile size (keeps query performance predictable)
            path,
            f"{schema}.{table}",
        ]

        db = settings.database
        psql_cmd = (
            f"PGPASSWORD={db.password} psql "
            f"-h {db.host} -p {db.port} -U {db.user} -d {db.dbname}"
        )

        import subprocess, shlex
        raster2pgsql = subprocess.run(cmd, capture_output=True, text=True)
        if raster2pgsql.returncode != 0:
            raise RuntimeError(
                f"raster2pgsql failed: {raster2pgsql.stderr}"
            )

        psql = subprocess.run(
            shlex.split(psql_cmd),
            input=raster2pgsql.stdout,
            capture_output=True,
            text=True,
        )
        if psql.returncode != 0:
            raise RuntimeError(
                f"psql import of shade raster failed: {psql.stderr}"
            )

        log(2, f"Shade coverage raster loaded → {schema}.{table}")
        logEndTask()


def create_shade_coverage_step(settings: GlobalSettings) -> ShadeCoverageStep:
    """Factory function - consistent with create_dem_step() pattern."""
    return ShadeCoverageStep(settings)
```

### 6. `core/attributes_step.py` - call the shade SQL template

In `run_step()`, add the following block alongside the other optional attribute
derivation calls (e.g. gradient, greenness, water):

```python
# --- Shade coverage ----------------------------------------------------------
if settings.optional.shade_coverage_path:
    log(2, "Deriving shade_coverage attribute...")
    logBeginTask()
    db.execute_template_sql_from_file(
        "sql/templates/attribute_shade_coverage.sql.j2",
        {
            "edge_table": f"{case_schema}.{edge_table}",
            "schema":     data_schema,           # netascore_data
        },
    )
    logEndTask()
else:
    log(3, "shade_coverage_path not set - shade_coverage column will be absent.")
```

### 7. `examples/settings_osm_query.yml` (and `settings_osm_file.yml`) - add optional key

Under the existing `optional:` section, add:

```yaml
optional:
  dem_path: data/bengaluru.tif          # existing example
  # ... other existing optional paths ...

  # Meta/WRI 1m Global Canopy Height - clipped to city extent
  # Export from GEE as single-band GeoTIFF, values 0–31 (metres), EPSG matching srid
  # Citation: Tolan et al. (2024), Remote Sensing of Environment, 300, 113888
  shade_coverage_path: data/bengaluru_canopy_height.tif
```

---

## Data preparation (GEE export)

Export the canopy height raster from Google Earth Engine before running the pipeline:

```javascript
var canopy_ht = ee.ImageCollection("projects/sat-io/open-datasets/facebook/meta-canopy-height");

// Define your city boundary
var city = ee.FeatureCollection("your_city_asset_or_geometry");

// Mosaic tiles, clip to city extent
var canopy_clipped = canopy_ht.mosaic().clip(city);

// Export as GeoTIFF - match SRID used in settings.yml
Export.image.toDrive({
  image: canopy_clipped,
  description: "bengaluru_canopy_height",
  scale: 1,                          // native 1m resolution
  region: city.geometry(),
  crs: "EPSG:32643",                 // UTM Zone 43N for Bengaluru; adjust per city
  maxPixels: 1e13,
  fileFormat: "GeoTIFF"
});
```

Place the exported `.tif` in `data/` and point `shade_coverage_path` to it in your
settings YAML.

---

## Performance note

`ST_PixelAsPoints` on a 1m raster generates ~600–800 pixel rows per edge within a
10m buffer. With 300k–400k edges (Delhi/Bengaluru scale) this is a heavy query. To
avoid timeouts, run the `attributes_step` UPDATE in batches:

```sql
-- Run in psql directly if needed, in chunks of 10,000 edges
UPDATE edge SET shade_coverage = ... WHERE edge_id BETWEEN 0 AND 9999;
UPDATE edge SET shade_coverage = ... WHERE edge_id BETWEEN 10000 AND 19999;
-- etc.
```

Alternatively, ensure a btree index on `edge_id` exists before the step runs -
NetAScore's `network_step` should already create this.

---

## Citation

```
Tolan, J., Yang, H.I., Nosarzewski, B., et al. (2024).
Very high resolution canopy height maps from RGB imagery using self-supervised
vision transformer and convolutional decoder trained on aerial lidar.
Remote Sensing of Environment, 300, 113888.

High Resolution Canopy Height Maps by WRI and Meta. Meta and World Resources
Institute (WRI) - 2023. Source imagery © 2016 Maxar.
License: Creative Commons Attribution 4.0 International (CC BY 4.0).
```