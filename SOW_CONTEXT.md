# Statement of Work Context — NetAScore India Adaptation

This document captures the project scope, technical decisions, and research findings behind the
effort to adapt NetAScore for Indian cities. It should be read alongside `REPOSITORY_CONTEXT.md`.

---

## Project Goal

Adapt the NetAScore open-source toolbox to produce valid, locally-calibrated walkability and
bikeability assessments for Indian cities — beginning with Bengaluru and Delhi — to support
evidence-based NMT (non-motorised transport) planning at the street-segment level.

The core challenge is that NetAScore's default indicator weights were validated in a European
(Salzburg) context and do not transfer directly to Indian OSM data without recalibration.

---

## Why This Matters

Most Indian cities lack granular, objective data on walking and cycling environment quality.
This adaptation enables planners and researchers to:

- Identify where NMT conditions are strong or critically weak
- Locate infrastructure gaps and discontinuities in the NMT network
- Prioritise street segments for upgrades based on evidence
- Inform 15-minute city and Transit-Oriented Development (TOD) assessments
- Track network quality over time as infrastructure investment is applied
- Support safe route mapping and least-stress routing for pedestrians and cyclists

---

## Pilot Cities & Data Already Processed

| City | Segments | OSM `.pbf` | DEM | Output `.gpkg` |
|---|---|---|---|---|
| Bengaluru | 322,159 | `bengaluru_region.osm.pbf` | `bengaluru.tif` (Bhuvan/ISRO) | `osm_network_bengaluru.gpkg` |
| Delhi | 387,590 | `delhi_region.osm.pbf` | `delhi.tif` | `osm_network_delhi.gpkg` |
| Mumbai | 87,007 | `mumbai_region.osm.pbf` | `mumbai.tif` | `osm_network_mumbai.gpkg` |

Validation: outputs for Bengaluru (Raj Bhavan Road) and Delhi (Lal Bahadur Shastri Marg)
were cross-referenced against Google Street View.

---

## Key Finding: OSM Coverage Gap

Shannon entropy weighting applied to segment-level indicator scores across all three cities
revealed a structural problem with the default NetAScore model in the Indian data context:

### Bike Profile
- `bicycle_infrastructure` (highest-weighted default indicator, w=0.20) → **zero entropy in
  all three cities** — absent on 99.8% of segments
- `designated_route` → absent on **100%** of segments
- These are pan-Indian OSM tagging gaps, not city-specific anomalies
- Three *inactive* indicators — `number_lanes`, `greenness`, `facilities` — captured
  **93–98% of total entropy weight**, meaning they are doing all the discriminating work

### Walk Profile
- `pedestrian_infrastructure` (highest default weight, w=0.40) → **zero entropy** across all
  cities; sidewalk tagged as present on 68–85% of segments without quality/width detail
- `noise` → 100% null everywhere (data provision gap, not a tagging issue)
- Active informative indicators: `water`, `crossings`, `number_lanes` (65–70% of entropy
  weight combined)
- Pearson correlation between entropy-derived index and original NetAScore walk index: 0.36–0.49

### Tagging Audit Summary

| Indicator | Bengaluru absent | Delhi absent | Mumbai absent |
|---|---|---|---|
| `bike_infra` | 99.8% | 99.8% | 99.8% |
| `ped_infra` | 78.6% | 85.3% | 68.7% |
| `designated_route` | 99.8% | 100% | 100% |
| `noise` | 100% | 100% | 100% |
| `pavement` | 79.9% | 85.6% | 90.6% |

**Conclusion:** The default weights do not transfer. Recalibration using locally-grounded
methods is required before results are meaningful for Indian cities.

---

## Adapted Indicator Set (11 Indicators)

The following indicators are selected for the India-adapted profiles (walk and bike):

1. Pedestrian Infrastructure / Bicycle Infrastructure
2. Road Type (`road_category`)
3. Speed Limit (`max_speed`)
4. Crossings Nearby (`crossings`)
5. Path Width (`width`)
6. **Shade Coverage** *(new — tree canopy + flyover shade; critical for Indian climate)*
7. Slope / Gradient (`gradient`, DEM-based via Bhuvan)
8. Green Spaces (`greenness`)
9. Water Bodies (`water`)
10. Nearby Facilities (`facilities`)
11. **Lighting Conditions** *(new)*

Proposed additional indicators (pending data availability):
- **Encroachment** — detectable via computer vision on Google Street View imagery
- **Network Continuity** — explicit graph-connectivity scoring (not just segment-level)
- **Pavement Absence as hard zero** — rather than a low score, to reflect binary barrier to use

---

## Weight Calibration Strategy

### Phase 1 (complete): Modified AHP (MAHP) Survey Results

**Why MAHP over traditional AHP:**
- Traditional AHP (Saaty 1980) requires n(n−1)/2 = 45 pairwise comparisons per mode
  (90 total), causing survey fatigue and 50–75% inconsistent responses in online deployment
- MAHP (Pascoe et al. 2024) replaces pairwise comparisons with a single-pass 1–9 rating
  table per mode — 10 ratings per mode, 20 total
- The pairwise comparison matrix is derived mathematically from score differences
- Demonstrated near-zero Geometric Consistency Index (GCI < 0.370 threshold) vs.
  near-universal inconsistency in equivalent online AHP surveys
- Statistically equivalent weights to traditional AHP (paired t-test, p > 0.05)

**Survey instrument:** KoboToolbox XLSForm, 1–9 Likert-style rating table per mode  
**Target respondents:** Urban planners, municipal engineers, cycling advocates, public health
professionals, academics across Indian cities

#### Walk Profile — MAHP Weights (India-calibrated)

| Rank | Indicator | MAHP Weight |
|---|---|---|
| 1 | Pedestrian infrastructure | 0.1562 |
| 2 | Crossings nearby | 0.1303 |
| 3 | Path width | 0.1171 |
| 4 | Speed limit | 0.1137 |
| 5 | Shade coverage | 0.0999 |
| 6 | Road type | 0.0898 |
| 7 | Nearby facilities | 0.0889 |
| 8 | Green space | 0.0886 |
| 9 | Slope / gradient | 0.0727 |
| 10 | Water bodies | 0.0428 |

Key shifts vs. default NetAScore walk profile:
- **Crossings nearby** rises sharply (ranked 2nd) — reflects Indian road-crossing safety concerns
- **Path width** enters top 3 — pavement width as a real discriminator in Indian urban streets
- **Shade coverage** (new indicator) ranks 5th — confirms importance of thermal comfort in Indian climate
- **Water bodies** drops to last — lower relevance in dense urban Indian contexts
- **Pedestrian infrastructure** remains #1 but at a much lower weight (0.156 vs. 0.40 default) —
  accounts for the fact that the indicator has near-zero discriminating power in current OSM data

#### Bike Profile — MAHP Weights (India-calibrated)

| Rank | Indicator | MAHP Weight |
|---|---|---|
| 1 | Surface quality | 0.1319 |
| 2 | Road type | 0.1238 |
| 3 | Speed limit | 0.1217 |
| 4 | Path / road width | 0.1173 |
| 5 | Cycling infrastructure | 0.1055 |
| 6 | Slope / gradient | 0.0904 |
| 7 | Shade coverage | 0.0892 |
| 8 | Green space | 0.0890 |
| 9 | Nearby facilities | 0.0794 |
| 10 | Water bodies | 0.0519 |

Key shifts vs. default NetAScore bike profile:
- **Surface quality** ranks 1st — potholes and unpaved surfaces are the primary barrier for cyclists in India
- **Cycling infrastructure** drops to 5th (vs. highest weight in default) — consistent with OSM tagging
  gap finding; expert survey acknowledges it matters but it is not the dominant factor in Indian reality
- **Shade coverage** (new indicator) ranks 7th — validates inclusion for cycling comfort in Indian climate
- **Road type** and **speed limit** both rank high (2nd/3rd) — traffic environment dominates where
  dedicated cycling infrastructure is absent
- Weights are notably more uniform across indicators (range: 0.052–0.132) vs. the default model's
  more concentrated weighting — reflects the multi-factor nature of Indian cycling conditions

#### Implementation Note for Profile YAML Files

These MAHP weights should be used directly in `examples/profile_walk_india.yml` and
`examples/profile_bike_india.yml`. NetAScore normalises weights internally, so raw MAHP
weights can be passed as-is under the `weight:` key for each indicator. Surface quality maps
to the existing `pavement` attribute in the codebase. Shade coverage requires a new
optional data layer and a new attribute column.

### Phase 2 (future): Shannon Entropy-Based Weighting

For use once OSM coverage improves sufficiently. Entropy-based weighting assigns weight
proportional to how much an indicator actually varies across segments — indicators that are
near-constant get downweighted automatically. To be combined with AHP weights using a
hybrid approach: AHP restores conceptual validity to data-sparse-but-important indicators;
entropy governs weights where data coverage is adequate.

---

## OSM Enrichment Strategy

Before or alongside recalibration, targeted OSM tagging enrichment is required for:

| Priority | Indicator | Gap | Approach |
|---|---|---|---|
| Critical | `bicycle_infrastructure` | ~100% absent | Field survey + MapRoulette campaigns |
| Critical | `designated_route` | 100% absent | Route digitisation from city plans |
| High | `pavement` | 80–91% absent | Field survey + StreetComplete app |
| High | `pedestrian_infrastructure` | quality/width missing | Attribute enrichment surveys |
| Low | `noise` | 100% null | External noise model integration |

---

## Planned Outputs

1. **Segment-level suitability scores (0–1)** — walk and bike, directional (`_ft` / `_tf`)
2. **Indicator breakdowns per segment** — diagnose *why* a segment scores poorly
3. **Bikeable Network Ratio (BNR) / Walkable Network Ratio (WNR)** — aggregated per
   pincode/ward (good segment length ÷ total street length, threshold ≥ 0.6); a single
   trackable KPI for municipal officials
4. **Critical improvement corridors** — ranked low-scoring segments connecting high-scoring
   clusters (phased investment prioritisation)
5. **Gap identification / missing links** — Steiner tree approximation on the road graph to find
   minimum-cost subgraphs that would connect disconnected good-network clusters
6. **Safe route maps** — GeoPackage outputs for GIS integration and planning tools

---

## Extended Feature Ideas (Annexure)

### Idea 1 — Recursive Scoring (Adjacent Link Propagation)
A segment's score is influenced by the quality of its neighbours (a good segment isolated in a
poor network is less useful). Implementation: graph diffusion / label propagation over the OSM
network using NetworkX. Run 2–3 iterations until convergence. Neighbour weight: 10–20% of
final score to avoid inflation in dense good-network areas.

### Idea 2 — Network Quality Ratio (V1-ready)
Aggregate BNR/WNR at pincode level → map showing which neighbourhoods are well-served
vs. underserved. One number per ward; trackable over time as infrastructure improves.
Directly relevant to municipal reporting needs.

### Idea 3 — Gap Identification (Highest-Impact Output)
- Identify good segments (score ≥ threshold) that are geographically close but disconnected
- Find shortest path of poor segments linking them — these are "missing links"
- Rank by: (length of poor segments needed) × (quality gain unlocked if gap is fixed)
- Output: prioritised intervention list — e.g. "fix 200m on Link Road → connects 4.2 km of
  good cycling infrastructure"
- Implementation: Steiner tree approximation via `networkx.algorithms.approximation.steiner_tree`

---

## Stakeholder Map

| Stakeholder | Use Case | NetAScore Output Needed |
|---|---|---|
| Urban Planners / Municipal Bodies (e.g., Dev Chaudhary IAS, MC Navsari) | Network gap analysis, street redesign, track upgrade impacts over time | GIS segment exports, BNR/WNR per ward |
| Academic / Research (e.g., Prof. Sandeep Paul, CEPT) | City benchmarking, AHP+entropy weight refinement, publications | Customisable indicator weights, raw segment data |
| Cycling Advocacy / Community (e.g., Chirag Shah, Kinjal Patel) | Identify unsafe corridors, lobby for infrastructure | Score maps, shareable route-level insights |
| Mobility Startups / App Developers (e.g., Nikita – Crooze; Arijit Soni – MyByks) | Comfort-based routing, user engagement | API integration of segment scores into routing engines |
| Transport Consultants (e.g., Sharad, UNITRANS) | 15-minute city assessments, mobility audits | City-wide analyses with locally adapted profiles |

---

## Next Steps

1. ~~Design and deploy MAHP survey instrument~~ **DONE** — weights derived for both profiles (see above)
2. Create `examples/profile_walk_india.yml` and `examples/profile_bike_india.yml` using the
   MAHP weights; wire in `surface_quality` (maps to `pavement`) and `shade_coverage` (new layer)
3. Add `shade_coverage` as a new optional importer in `core/optional_step.py` and a corresponding
   attribute in `core/attributes_step.py` + SQL templates
4. Run NetAScore with adapted profiles on Bengaluru or Delhi; conduct on-ground validation
   against user perception data
5. Implement BNR/WNR pincode-level aggregation as V1 reporting output
6. Pursue targeted OSM enrichment for highest-gap indicators (bike infra, pavement, ped infra quality)
7. Publish findings and release open datasets to support replication across other Indian cities

---

## References

- Werner, C. et al. (2024). NetAScore: An open and extendible software for segment-scale
  bikeability and walkability. *Environment and Planning B*, 52(1), 265–274.
- Werner, C. et al. (2024). Bikeability of road segments: An open, adjustable, and extendible
  model. *Journal of Cycling and Micromobility Research*, 2, 100040.
- Stutz, P. et al. (2025). Walkability at Street Level: An Indicator-Based Assessment Model.
  *Sustainability*, 17(8), 3634.
- Loidl, M., & Zagel, B. (2014). Assessing bicycle safety in multiple networks with different
  data models. *GI_Forum*, 2, 144–154.
- Pascoe, S. et al. (2024). Modified AHP (MAHP) — near-zero GCI, statistically equivalent
  weights, higher completion rates.
- Melillo & Pecchia (2016). Minimum sample size of 19 for AHP-derived weight validity.
- Shannon, C. (1948). A mathematical theory of communication. *Bell System Technical Journal*.
- Zou, Z. et al. (2006). Entropy-based weighting in multi-criteria assessment.
