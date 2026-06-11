# NetAScore - Network Assessment Score Toolbox for Sustainable Mobility (India Adaptation)

<picture>
  <source media="(prefers-color-scheme: dark)" srcset="https://github.com/plus-mobilitylab/netascore/assets/82904077/762dc210-1ca5-4ead-8aeb-522e974a93fe">
  <source media="(prefers-color-scheme: light)" srcset="https://github.com/plus-mobilitylab/netascore/assets/82904077/240d09f8-a728-41ec-b0e7-8bba8fac4d38">
  <img alt="NetAScore logo" src="https://github.com/plus-mobilitylab/netascore/assets/82904077/240d09f8-a728-41ec-b0e7-8bba8fac4d38">
</picture>

This is an adaptation of [NetAScore](https://github.com/plus-mobilitylab/netascore) for the Indian context. It computes **bikeability** and **walkability** indicators from **OpenStreetMap (OSM)** data for cities and regions across India.

NetAScore provides an automated workflow for assessing infrastructure suitability for cycling (*bikeability*) and walking (*walkability*) at the street-segment level. By editing settings files and mode profiles, assessments can be customised to reflect local conditions, infrastructure types, and user preferences relevant to the Indian urban environment.

For citing NetAScore, please refer to the following paper:

Werner, C., Wendel, R., Kaziyeva, D., Stutz, P., van der Meer, L., Effertz, L., Zagel, B., & Loidl, M. (2024). NetAScore: An open and extendible software for segment-scale bikeability and walkability. *Environment and Planning B: Urban Analytics and City Science*, 0(0). [https://doi.org/10.1177/23998083241293177]

Details on the software implementation are archived at [doi.org/10.5281/zenodo.7695369](https://doi.org/10.5281/zenodo.7695369).

---

## India-specific Context

Indian cities present a distinct mobility landscape: dense mixed-use streets, high pedestrian volumes, significant cycling activity (especially for utilitarian trips), and varying infrastructure quality across urban and peri-urban areas. OSM coverage in major Indian cities (Delhi, Mumbai, Bengaluru, Chennai, Hyderabad, Pune, and others) is generally sufficient for meaningful bikeability and walkability assessments, though coverage varies by locality.

### Notes on OSM Data Quality in India

- **Major metros** (Delhi, Mumbai, Bengaluru, etc.) typically have good road network coverage. Footpath, cycle lane, and surface-type tagging is improving but may be incomplete in many areas.
- **Tier-2 and Tier-3 cities** may have sparser attribute data. Results should be interpreted with awareness of local tagging completeness.
- **Missing attributes** (e.g. surface type, footway presence) will cause NetAScore to fall back to defaults or mark segments as having insufficient data. We recommend reviewing the OSM data coverage for your area of interest before running an assessment.
- You can check and improve OSM data for your area via [OpenStreetMap.org](https://www.openstreetmap.org) or tools like [MapRoulette](https://maproulette.org) and [StreetComplete](https://streetcomplete.app).

### Recommended Indicators for Indian Cities

The default bikeability and walkability presets work with Indian OSM data. However, the following local considerations are worth noting when interpreting or customising results:

- **Footpath availability** is a key walkability driver in Indian cities and is increasingly tagged in OSM (`footway`, `sidewalk` tags).
- **Surface quality** (`surface` tag) and **road category** (`highway` tag) are the most reliably available attributes for bikeability scoring in India.
- **Traffic volume proxies** (derived from road category) are used where direct traffic count data is unavailable, which is typical for OSM-based assessments.
- **Shade and greenery** along streets, important for pedestrian comfort in India's climate, can be incorporated via the `shade_coverage` optional indicator using the Meta/WRI 1m Global Canopy Height raster (see [Shade Coverage](#shade-coverage-optional) below).

---

## How to get started?

### Easy quickstart: ready-made Docker image

The easiest way to get started is running the ready-made Docker image. You need [Docker](https://docs.docker.com/engine/install/) installed and running, plus an internet connection. Then follow these two steps:

1. Download the `docker-compose.yml` file from the `examples` folder ([download raw file](https://github.com/plus-mobilitylab/netascore/blob/main/examples/docker-compose.yml)) into an empty directory.
2. Open a terminal in that directory and run:

```
docker compose run netascore
```

By default this runs the example case for Salzburg, Austria. **This India adaptation uses a local OSM file as input** - see the next section for how to configure and run it for your area.

### Running an assessment for an Indian city

This adaptation uses **`settings_osm_file.yml`** as the primary settings file, which reads a locally downloaded OSM file rather than querying the Overpass API at runtime. This is recommended for Indian cities because:

- It gives you full control over the OSM extract (date, bounding box, completeness).
- It avoids Overpass API timeouts for large or complex areas.
- It allows you to pre-process or enrich the OSM data before running the assessment.

**Step 1 - Download an OSM extract for your area**

Download a `.osm.pbf` file for your city or region from one of these sources:

- [Geofabrik India extracts](https://download.geofabrik.de/asia/india.html) - state-level and subregion extracts
- [BBBike extracts](https://extract.bbbike.org/) - custom bounding-box extracts for any city

For a specific city boundary, you can clip the state extract using `osmium`:

```bash
osmium extract --bbox=<min_lon,min_lat,max_lon,max_lat> india-latest.osm.pbf -o my_city.osm.pbf
```

**Step 2 - Configure `settings_osm_file.yml`**

Edit `settings_osm_file.yml` and set the path to your OSM file:

```yaml
osm_file: "data/my_city.osm.pbf"
```

Adjust the output filename and any other parameters as needed.

**Step 3 - Run NetAScore**

```bash
docker compose run netascore data/settings_osm_file.yml
```

Or, if running locally with Python, pass the settings file explicitly:

```bash
python generate_index.py data/settings_osm_file.yml
```

> **Tip:** For large metros such as Delhi or Mumbai, clip to a district or planning zone before processing to keep memory usage and runtime manageable.

---

## Shade Coverage (Optional)

This adaptation adds **`shade_coverage`** as an optional indicator - tree canopy and overhead shade are among the most significant factors for walking and cycling willingness in India's climate.

**Data source:** [Meta/WRI 1m Global Canopy Height Maps](https://gee-community-catalog.org/projects/meta_trees/) (Tolan et al. 2024), available free via Google Earth Engine.

**How it works:** For each road segment, pixels from the canopy height raster within a 10 m buffer are scored on a height-weighted curve (0 m → 0.0, ≥18 m → 1.0) and averaged to produce a segment-level shade score in [0, 1].

**To enable it:**

1. Export the canopy height raster for your city from GEE:

```javascript
var canopy = ee.ImageCollection("projects/sat-io/open-datasets/facebook/meta-canopy-height")
               .mosaic().clip(city_geometry);
Export.image.toDrive({
  image: canopy,
  description: "my_city_canopy_height",
  scale: 1,
  crs: "EPSG:4326",
  maxPixels: 1e13,
  fileFormat: "GeoTIFF"
});
```

2. Place the exported `.tif` in `data/` and add to your settings file:

```yaml
optional:
  shade_coverage:
    filename: my_city_canopy_height.tif
    srid: 4326
```

3. Use the India-adapted mode profiles which include `shade_coverage` pre-weighted:

```yaml
profiles:
  - profile_name: walk_india
    filename: profile_walk_india.yml
    filter_access_walk: True
  - profile_name: bike_india
    filename: profile_bike_india.yml
    filter_access_bike: True
```

**Citation:**
> Tolan, J., Yang, H.I., Nosarzewski, B., et al. (2024). Very high resolution canopy height maps from RGB imagery using self-supervised vision transformer and convolutional decoder trained on aerial lidar. *Remote Sensing of Environment*, 300, 113888.

---

## India-adapted Mode Profiles

This adaptation includes two MAHP-calibrated mode profiles derived from a structured expert survey (KoboToolbox, n=25–35, urban planning practitioners across Indian cities):

| Profile file | Mode | Key differences from default |
|---|---|---|
| `profile_walk_india.yml` | Walking | Crossings (rank 2), path width (rank 3), shade (rank 5); water bodies deprioritised |
| `profile_bike_india.yml` | Cycling | Surface quality is rank 1; cycling infrastructure drops to rank 5 (OSM coverage gap) |

See [`SOW_CONTEXT.md`](SOW_CONTEXT.md) for full methodology and weight derivation details.

---

## What the output contains

After a successful run, a `data/` subdirectory will be created. The assessed network is stored as a GeoPackage (`.gpkg`) file. It includes:

| Column | Description |
|---|---|
| `index_bike_ft` | Bikeability score, forward direction (0 = unsuitable, 1 = well-suited) |
| `index_bike_tf` | Bikeability score, reverse direction |
| `bikeability` | Segment-level bikeability: `max(index_bike_ft, index_bike_tf)` |
| `index_walk_ft` | Walkability score, forward direction |
| `index_walk_tf` | Walkability score, reverse direction |
| `walkability` | Segment-level walkability: `max(index_walk_ft, index_walk_tf)` |

`ft` = *from-to* node direction; `tf` = *to-from* node direction.

> **Bikeability score:** The reported per-segment bikeability is derived as `max(index_bike_ft, index_bike_tf)` - the better of the two directional scores. This reflects that a cyclist can travel in whichever direction is more suitable along a segment, making the maximum the appropriate summary statistic for segment-level analyses and mapping.

> **Walkability score:** The reported per-segment walkability is derived as `max(index_walk_ft, index_walk_tf)` - the better of the two directional scores.

### Visualising results in QGIS

NetAScore does not include a built-in visualisation module. To view results:

1. Install [QGIS](https://qgis.org) (free and open-source).
2. Drag and drop the `.gpkg` file into a new QGIS project.
3. Select the `edge` layer.
4. In layer properties, define a graduated colour symbology on `bikeability` or `walkability`.

A value of `0` means the segment is assessed as unsuitable; `1` means well-suited infrastructure.

---

## Running NetAScore locally (without Docker)

For running NetAScore without Docker, you need several software packages and Python libraries. Full instructions are in the [wiki](https://github.com/plus-mobilitylab/netascore/wiki/How-to-run-the-project).

**NetAScore uses the following technologies:**

- Python 3
- PostgreSQL with PostGIS extension
- Docker (optional)
- psql, ogr2ogr, osm2pgsql, raster2pgsql
- [Several Python libraries](requirements.txt)

---

## More information

Full documentation is available in the **[NetAScore wiki](https://github.com/plus-mobilitylab/netascore/wiki)**:

- [About NetAScore](https://github.com/plus-mobilitylab/netascore/wiki)
- [Quickstart Guide](https://github.com/plus-mobilitylab/netascore/wiki/Quickstart%E2%80%90Guide)
- [The Workflow](https://github.com/plus-mobilitylab/netascore/wiki/The-workflow)
- [Running in Docker](https://github.com/plus-mobilitylab/netascore/wiki/How-to-run-the-project-in-a-Docker-environment)
- [Running with Python](https://github.com/plus-mobilitylab/netascore/wiki/Run-NetAScore-manually-with-Python)
- [Attributes & Indicators](https://github.com/plus-mobilitylab/netascore/wiki/Attributes-and-Indicators)
- [Attribute derivation from OSM](https://github.com/plus-mobilitylab/netascore/wiki/Attribute-derivation-from-OSM)
- [Configuration of Settings](https://github.com/plus-mobilitylab/netascore/wiki/Configuration-of-the-settings)
- [Requirements and Limitations](https://github.com/plus-mobilitylab/netascore/wiki/Requirements-and-Limitations)
- [Credits and License](https://github.com/plus-mobilitylab/netascore/wiki/Credits-and-license)

Example output files are available at [doi.org/10.5281/zenodo.10886961](https://doi.org/10.5281/zenodo.10886961).

---

## Contributing

Contributions to improve this India adaptation are very welcome - particularly around:

- Refined indicator weights reflecting Indian road conditions
- Validation studies for Indian cities
- Guidance on integrating municipal GIS data (e.g. from Smart Cities Mission portals)

See the [contribution guide](https://github.com/plus-mobilitylab/netascore/wiki/How-to-contribute) for details.

---

## License

This project is licensed under the MIT License - see [LICENSE](LICENSE) for details.

The upstream NetAScore project is developed by the [PLUS Mobility Lab](https://github.com/plus-mobilitylab), University of Salzburg, and is also MIT licensed.
