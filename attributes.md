# Attributes / Indicators

Attributes / Indicators describe a certain property of a road segment.
Attributes / Indicators indicated with an asterisk (*) are differentiated by direction (ft/tf).
Indicators are used for index calculation.   

## Attributes

### access_car\_* / access_bicycle\_* / access_pedestrian\_*

Indicates the accessibility of a road segment for cars, bicycles or pedestrians: `true`, `false`

### bridge

Indicates a bridge on a road segment: `true`, `false`

### tunnel

Indicates a tunnel on a road segment: `true`, `false`

## Indicators

### bicycle_infrastructure_*

Describes the existence of dedicated bicycle infrastructure: `bicycle_way`, `mixed_way`, `bicycle_road`, `cyclestreet`, `bicycle_lane`, `bus_lane`, `no`.

### buildings

Describes the proportion of the area of buildings within a 30 meters buffer: `0` to `100`.


### crossings

Describes the amount of crossings within a 10 meters buffer.


### designated_route_*

Describes the existence of designated cycling routes categorized by impact: `local`, `regional`, `national`, `international`, `unknown`, `no`.


### facilities

Describes the amount of facilities (POIs) within a 30 meters buffer.


### gradient_*

Describes the gradient class of a road segment for downhill and uphill: `-4` to `4`.

| Class | Definition  |
|-------|-------------|
| 0     | 0 - 1,5 %   |
| 1     | > 1,5 - 3 % |
| 2     | > 3 - 6 %   |
| 3     | > 6 - 12 %  |
| 4     | > 12 %      |

The influence of gradient classes on the final index can be assigned per mode using the section `indicator_mapping` within mode profile files.


### greenness

Describes the proportion of the green area within a 30 meters buffer: `0` to `100`.


### max_speed\_* / max_speed_greatest

`max_speed_*` describes the speed limit (car) in the direction of travel or the average speed (car), if speed limit is not available: `0` to `130`. `max_speed_greatest_*` uses the maximum value of speed limits for both directions of travel on this segment.


### noise

Describes the noise level of a road segment in decibel.


### number_lanes_*

Describes the number of lanes of a road segment.


### parking_* (not in use)

Describes designated parking lots: `yes`, `no`. Currently, this indicator is not computed due to data availability. This will be documented accordingly in the `index_<mode>_*_robustness`-column in the output dataset.


### pavement

Describes the condition of the road surface: `asphalt`, `gravel`, `cobble`, `soft`.


### pedestrian_infrastructure_*

Describes the existence of dedicated pedestrian infrastructure: `pedestrian_area`, `pedestrian_way`, `mixed_way`, `stairs`, `sidewalk`, `no`.


### road_category

Describes the road category of a road segment: `primary`, `secondary`, `residential`, `service`, `calmed`, `no_mit`, `path`.


### water

Describes the occurrence of water bodies within a 30 meters buffer: `true`, `false`.


### width

Describes the width class of a road segment derived from the OSM `width` tag after cleaning.
**Represents total Right-of-Way (ROW) or shared carriageway width — not the width of any
dedicated footpath or cycle track.**

| Class | Total ROW       | Walking feasibility (IRC 103)                            | Cycling feasibility (IRC 11)                          |
|-------|----------------|----------------------------------------------------------|-------------------------------------------------------|
| 0     | < 5 m           | No footpath possible within ROW                          | No dedicated space; cyclists mix with all traffic     |
| 1     | 5 – 10 m        | IRC 103 residential footpath theoretical but rarely achieved | Below IRC 11 minimum (2.0 m) for dedicated track  |
| 2     | 10 – 20 m       | IRC 103 residential footpath (1.8 m clear zone) feasible | IRC 11 Type B painted lane (2.0 m) alongside carriageway |
| 3     | 20 – 35 m       | IRC 103 commercial footpath (2.5 m zone) feasible         | IRC 11 Type A segregated track (2.5 m) feasible      |
| 4     | > 35 m          | Full three-zone high-intensity footpath or promenade      | Bidirectional cycle track (3.0–3.5 m) with buffer    |

Width is used as a **proxy** for the likely available space for walking and cycling: wider ROW
correlates with greater feasibility of IRC-compliant footpath (IRC 103:2012) and cycle track
(IRC 11:1962, amended 2022) provision. It is a **non-directional** indicator (no `ft`/`tf` suffix).
Segments with no OSM `width` tag will have a NULL value and are reflected in the
`index_<mode>_*_robustness` output column.
