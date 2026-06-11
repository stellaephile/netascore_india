# Contributing to NetAScore India Adaptation

Thank you for your interest in contributing. This project adapts [NetAScore](https://github.com/plus-mobilitylab/netascore) for Indian cities — contributions that improve its accuracy, coverage, and usability in the Indian context are especially welcome.

---

## Ways to Contribute

### 1. Indicator weight refinement
The current walk and bike profiles use MAHP-derived weights from an initial expert survey. If you have domain expertise in Indian urban mobility, public health, or transport planning, you can contribute by:
- Participating in or running a new MAHP/AHP weighting survey for a specific city or user group
- Proposing evidence-based adjustments to indicator value mappings (e.g. speed thresholds, width breakpoints) based on Indian conditions
- Sharing results from perception studies or user surveys that could validate or challenge current weights

### 2. Validation studies
We have run preliminary validation for Bengaluru (Raj Bhavan Road) and Delhi (Lal Bahadur Shastri Marg) using Google Street View. Contributions welcomed:
- Ground-truth comparisons between NetAScore outputs and on-street conditions
- Correlation of scores against user perception data (surveys, crowdsourced ratings)
- City-specific validation reports with reproducible methodology

### 3. New cities and datasets
- Settings files and DEM data for additional Indian cities
- Pre-processed OSM extracts or enriched data for cities with known tagging gaps
- Integration guidance for municipal GIS datasets (Smart Cities Mission portals, city open data)

### 4. OSM data improvement
Many Indian cities have gaps in OSM tagging for attributes NetAScore relies on (bicycle infrastructure, surface type, pedestrian infrastructure quality). You can help by:
- Contributing OSM edits for your city via [OpenStreetMap.org](https://www.openstreetmap.org)
- Running [StreetComplete](https://streetcomplete.app) surveys to fill attribute gaps
- Creating [MapRoulette](https://maproulette.org) challenges for systematic tagging campaigns

### 5. Code contributions
Bug fixes, performance improvements, and new features are welcome. Priority areas:
- **Bikeable/Walkable Network Ratio (BNR/WNR)** — pincode/ward-level aggregation of segment scores into a single reportable KPI (see `SOW_CONTEXT.md`)
- **Recursive scoring** — graph diffusion to propagate scores from high-quality to adjacent segments
- **Gap identification** — Steiner tree based detection of missing links between high-scoring clusters
- **Shade coverage** improvements — batch processing for large cities, alternative canopy data sources
- **New optional indicators** — encroachment detection, lighting conditions

---

## Getting Started

1. **Fork** this repository and clone your fork locally.
2. Set up the development environment using Docker (see `README.md`).
3. Create a new branch for your change:
   ```bash
   git checkout -b your-feature-name
   ```
4. Make your changes, test them against at least one Indian city dataset.
5. Open a pull request with a clear description of what changed and why.

---

## Code Style and Standards

- Follow the existing module structure: one class per pipeline step, factory functions for instantiation, `DbStep` as base class.
- New optional indicators should follow the `DemImporter` / `ShadeCoverageImporter` pattern in `core/optional_step.py`.
- New SQL logic should be in a Jinja2 template under `sql/templates/`, not inline Python strings.
- Use `toolbox/helper.py` logging functions (`h.info`, `h.log`, `h.logBeginTask`) — no bare `print()` calls.
- Settings for new optional layers go in the `optional:` section of the YAML settings file; document them in `settings.md`.

---

## Reporting Issues

If you find a bug, unexpected output, or data quality issue:

1. Check whether the issue is specific to Indian OSM data (tagging gap) or a code bug.
2. Open a GitHub issue with:
   - The city and bounding box you were processing
   - The settings file used (redact any credentials)
   - The full error traceback or description of unexpected output
   - Steps to reproduce

---

## Upstream Attribution

This project is a downstream adaptation of [NetAScore](https://github.com/plus-mobilitylab/netascore) by the PLUS Mobility Lab, University of Salzburg. Contributions that are general-purpose (not India-specific) are encouraged to be submitted upstream as well.

Please cite the original software if you publish work using this adaptation:

> Werner, C., et al. (2024). NetAScore: An open and extendible software for segment-scale bikeability and walkability. *Environment and Planning B*, 52(1), 265–274. https://doi.org/10.1177/23998083241293177

---

## License

By contributing, you agree that your contributions will be licensed under the [MIT License](LICENSE).
