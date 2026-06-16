# Sydney Station Urban Context Data

This project builds station-level urban context features for 295 permanent traffic monitoring stations in Greater Sydney. For each station, the workflow creates a 500 metre buffer and extracts nearby OpenStreetMap-derived indicators for points of interest, buildings, and land use.

The final output is a tidy station-level dataset that can be joined with traffic or mobility modelling data.

## Project Overview

The processing workflow uses station coordinates and an OpenStreetMap `.osm.pbf` extract for Sydney to generate three groups of spatial features:

- **POI accessibility features**: counts of food, education, healthcare, public transport, leisure, tourism, and shop POIs within 500 metres.
- **Building intensity features**: building count and building density per square kilometre within 500 metres.
- **Land use composition features**: percentage share of commercial, green/recreation, industrial, institutional, residential, and transport/other land use around each station.

All spatial calculations use projected coordinates in `EPSG:7856`, which is suitable for metre-based distance and area calculations around Sydney.

## Repository Structure

```text
.
|-- sydney_permanent_stations_candidate_1.csv
|-- notebooks/
|   |-- 01_station_candidate_selection.ipynb
|   `-- 02_urban_context_feature_extraction.ipynb
|-- data/
|   `-- osm/
|       `-- Sydney.osm.pbf
`-- POIs_output/
    |-- sydney_permanent_stations_clean.csv
    |-- sydney_permanent_stations_clean.gpkg
    |-- station_poi_features_500m.csv
    |-- station_building_features_500m.csv
    |-- station_landuse_features_500m.csv
    |-- sydney_station_urban_context_500m.csv
    `-- sydney_station_urban_context_500m.gpkg
```

The cleaned notebooks in `notebooks/` are the recommended workflow for reproducing this project.

## Notebook Workflow

Run the notebooks in the following order:

1. `notebooks/01_station_candidate_selection.ipynb`
   - Reads `road_traffic_counts_station_reference.csv`.
   - Selects published Sydney permanent monitoring stations with quality rating at least 4.
   - Exports `sydney_permanent_stations_candidate_1.csv`.

2. `notebooks/02_urban_context_feature_extraction.ipynb`
   - Reads the cleaned station candidate table and local Sydney OSM extract.
   - Creates 500m station buffers.
   - Extracts POI, building, and land-use features.
   - Exports the final station-level urban context dataset.

## Data Inputs

### Traffic-Station Candidate Selection

The spatial reference stations were derived from the Transport for NSW station metadata file:

```text
road_traffic_counts_station_reference.csv
```

This station-reference table is separate from the hourly traffic-count tables. It contains station identifiers, road and intersection names, regional information, station status, publication status, quality ratings, and WGS84 coordinates.

The candidate stations were selected using the following conditions:

```python
sydney_stations = df[
    (df["rms_region"] == "Sydney")
    & (df["permanent_station"] == 1)
    & (df["publish"] == True)
    & (df["quality_rating"] >= 4)
].copy()
```

A station was retained when:

- its Transport for NSW RMS region was recorded as `Sydney`
- it was classified as a permanent monitoring station
- the record was approved for publication
- its quality rating was at least 4

The following fields were retained:

```text
station_key
station_id
name
road_name
intersection
lga
suburb
post_code
device_type
wgs84_latitude
wgs84_longitude
```

The resulting table contained 295 monitoring stations and was exported as:

```text
sydney_permanent_stations_candidate_1.csv
```

This candidate table was subsequently used as:

- the station reference table for constructing 500m spatial buffers
- the set of station keys used to filter the hourly traffic-count records
- the join key for integrating POI, building, and land-use features

### OpenStreetMap Extract

`data/osm/Sydney.osm.pbf` is used as the local OSM source. The workflow reads OSM layers with `pyogrio`, extracts tags from `other_tags`, and avoids relying on repeated online Overpass API requests.

Because the `.osm.pbf` file is large, consider storing it outside GitHub or using Git LFS if the repository must include it.

## Processing Workflow

1. **Load station locations**
   - Read station CSV.
   - Convert station coordinates to a GeoDataFrame in `EPSG:4326`.
   - Reproject to `EPSG:7856` for metre-based buffers.

2. **Create 500m station buffers**
   - Generate a 500 metre buffer around each station.
   - Use these buffers as the spatial unit for all feature extraction.

3. **Extract and clean POIs**
   - Read OSM point and multipolygon layers.
   - Extract tags including `amenity`, `shop`, `public_transport`, `leisure`, `tourism`, and `healthcare`.
   - Convert polygon POIs to representative points.
   - Remove likely duplicates where named point and polygon POIs match within 20 metres.
   - Spatially join POIs to station buffers and aggregate category counts.

4. **Extract building features**
   - Select OSM polygons with a non-empty `building` tag.
   - Spatially join buildings to station buffers.
   - Count unique buildings per station.
   - Calculate building density as `building_count / (pi * 0.5^2)` km2.

5. **Extract land use features**
   - Select OSM polygons with a `landuse` tag.
   - Map detailed land use values into six broader categories.
   - Intersect land use geometries with each station buffer.
   - Calculate category percentages from mapped land use area.

6. **Merge final dataset**
   - Merge POI, building, and land use features back to the station table.
   - Save final outputs as CSV and GeoPackage.

## Output Files

### Final Dataset

`POIs_output/sydney_station_urban_context_500m.csv`

This is the main modelling-ready output. It contains 295 rows and 29 columns:

- station metadata
- 7 POI count features
- POI missing-data flag
- 2 building features
- building missing-data flag
- 6 land use percentage features
- land use missing-data flag

`POIs_output/sydney_station_urban_context_500m.gpkg`

GeoPackage version of the final dataset, preserving station geometry.

### Intermediate Outputs

| File | Rows | Columns | Description |
| --- | ---: | ---: | --- |
| `sydney_permanent_stations_clean.csv` | 295 | 11 | Cleaned permanent station table |
| `station_poi_features_500m.csv` | 295 | 18 | POI category counts within 500m |
| `station_building_features_500m.csv` | 295 | 4 | Building count and density within 500m |
| `station_landuse_features_500m.csv` | 295 | 17 | Land use category percentages within 500m |
| `sydney_station_urban_context_500m.csv` | 295 | 29 | Final merged station-level dataset |

## Data Dictionary

### Source Datasets

#### Transport for NSW Station Reference

Source file:

```text
road_traffic_counts_station_reference.csv
```

This file provides the station metadata used to define the spatial anchor points for the project.

| Field | Description |
| --- | --- |
| `station_key` | Unique station key used as the primary identifier across station and traffic-count datasets |
| `station_id` | Transport for NSW station identifier |
| `name` | Station or count-site name |
| `road_name` | Road where the traffic monitoring station is located |
| `intersection` | Nearby intersecting road or reference location |
| `rms_region` | Transport for NSW RMS region; used to retain Sydney stations |
| `lga` | Local Government Area |
| `suburb` | Suburb name |
| `post_code` | Postal code |
| `device_type` | Monitoring device type |
| `permanent_station` | Indicator for permanent monitoring stations |
| `quality_rating` | Station quality rating; this project retained records with rating at least 4 |
| `publish` | Publication flag; this project retained published records |
| `wgs84_latitude` | Latitude in WGS84 |
| `wgs84_longitude` | Longitude in WGS84 |

#### Transport for NSW Hourly Traffic Counts

The hourly traffic-count tables are separate from the station-reference metadata. They are not the source of the station coordinates, but the selected station keys can be used to filter traffic-count records for modelling or downstream analysis.

Typical fields include:

| Field | Description |
| --- | --- |
| `station_key` | Station key used to join traffic observations to station metadata |
| Date/time fields | Timestamp, date, hour, or interval fields describing when counts were observed |
| Direction or lane fields | Directional, lane, or carriageway attributes where available |
| Count fields | Observed traffic volume or class-specific traffic counts |

#### OpenStreetMap Sydney Extract

Source file:

```text
data/osm/Sydney.osm.pbf
```

This local OSM extract provides POI, building, and land-use geometries. The workflow reads the `points` and `multipolygons` layers and extracts relevant tags from either explicit columns or the OSM `other_tags` field.

| Tag or Field | Description |
| --- | --- |
| `osm_id` / `osm_way_id` | OSM object identifiers used to construct unique feature IDs |
| `name` | OSM feature name, when available |
| `amenity` | POI-related tag for services such as restaurants, schools, hospitals, parking, and public facilities |
| `shop` | Retail/shop category tag |
| `public_transport` | Public transport infrastructure tag |
| `leisure` | Leisure and recreation tag |
| `tourism` | Tourism-related tag |
| `healthcare` | Healthcare-specific tag |
| `building` | Building indicator or building type |
| `landuse` | Land-use category tag |
| `geometry` | OSM feature geometry |
| `other_tags` | Additional OSM tags stored as key-value strings in the PBF-derived layer |

### Candidate and Intermediate Datasets

#### `sydney_permanent_stations_candidate_1.csv`

This is the cleaned station candidate table used by both notebooks. It contains 295 Sydney permanent traffic monitoring stations.

| Field | Description |
| --- | --- |
| `station_key` | Unique station key used as the main join key |
| `station_id` | Transport for NSW station identifier |
| `name` | Station or site name |
| `road_name` | Road where the station is located |
| `intersection` | Nearby intersection or reference road |
| `lga` | Local Government Area |
| `suburb` | Suburb name |
| `post_code` | Postal code |
| `device_type` | Monitoring device type |
| `wgs84_latitude` | Station latitude in WGS84 |
| `wgs84_longitude` | Station longitude in WGS84 |

#### `POIs_output/station_poi_features_500m.csv`

POI category counts within each station's 500m buffer.

| Field | Description |
| --- | --- |
| Station metadata fields | `station_key`, `station_id`, `name`, `road_name`, `intersection`, `lga`, `suburb`, `post_code`, `wgs84_latitude`, `wgs84_longitude` |
| `poi_food_count_500m` | Count of food-related POIs, such as restaurants, cafes, fast food, pubs, bars, and food courts |
| `poi_education_count_500m` | Count of education POIs, such as schools, universities, colleges, kindergartens, and childcare facilities |
| `poi_healthcare_count_500m` | Count of healthcare POIs, such as hospitals, clinics, doctors, dentists, pharmacies, or features with healthcare tags |
| `poi_public_transport_count_500m` | Count of public transport POIs, such as platforms, stations, or stop positions |
| `poi_leisure_count_500m` | Count of leisure POIs, such as parks, gardens, playgrounds, pitches, sports centres, and fitness centres |
| `poi_tourism_count_500m` | Count of tourism POIs, such as hotels, attractions, museums, artworks, and information features |
| `poi_shop_count_500m` | Count of OSM features with a non-empty `shop` tag |
| `poi_data_missing` | `True` when no POI category was matched within the 500m buffer |

#### `POIs_output/station_building_features_500m.csv`

Building intensity features within each station's 500m buffer.

| Field | Description |
| --- | --- |
| `station_key` | Station join key |
| `building_count_500m` | Number of unique OSM building features intersecting the 500m buffer |
| `building_density_per_km2_500m` | Building count divided by the buffer area in square kilometres |
| `building_data_missing` | `True` when no building feature was matched within the 500m buffer |

#### `POIs_output/station_landuse_features_500m.csv`

Land-use composition features within each station's 500m buffer.

| Field | Description |
| --- | --- |
| Station metadata fields | `station_key`, `station_id`, `name`, `road_name`, `intersection`, `lga`, `suburb`, `post_code`, `wgs84_latitude`, `wgs84_longitude` |
| `commercial_pct_500m` | Percentage of mapped land-use area classified as commercial |
| `green_recreation_pct_500m` | Percentage of mapped land-use area classified as green or recreation space |
| `industrial_pct_500m` | Percentage of mapped land-use area classified as industrial |
| `institutional_pct_500m` | Percentage of mapped land-use area classified as institutional or public service land use |
| `residential_pct_500m` | Percentage of mapped land-use area classified as residential |
| `transport_other_pct_500m` | Percentage of mapped land-use area classified as transport, construction, cemetery, or other mapped land use |
| `landuse_data_missing` | `True` when no mapped land-use polygon intersected the 500m buffer |

### Final Processed Dataset

#### `POIs_output/sydney_station_urban_context_500m.csv`

This is the final modelling-ready table. Each row represents one traffic monitoring station and its surrounding 500m urban context.

| Field | Description |
| --- | --- |
| `station_key` | Unique station key and primary join field |
| `station_id` | Transport for NSW station identifier |
| `name` | Station or count-site name |
| `road_name` | Road where the station is located |
| `intersection` | Nearby intersection or reference road |
| `lga` | Local Government Area |
| `suburb` | Suburb name |
| `post_code` | Postal code |
| `device_type` | Monitoring device type |
| `wgs84_latitude` | Station latitude in WGS84 |
| `wgs84_longitude` | Station longitude in WGS84 |
| `poi_food_count_500m` | Count of food-related POIs within 500m |
| `poi_education_count_500m` | Count of education POIs within 500m |
| `poi_healthcare_count_500m` | Count of healthcare POIs within 500m |
| `poi_public_transport_count_500m` | Count of public transport POIs within 500m |
| `poi_leisure_count_500m` | Count of leisure POIs within 500m |
| `poi_tourism_count_500m` | Count of tourism POIs within 500m |
| `poi_shop_count_500m` | Count of shop-tagged POIs within 500m |
| `poi_data_missing` | `True` when all POI count features are zero |
| `building_count_500m` | Number of unique OSM building features intersecting the 500m buffer |
| `building_density_per_km2_500m` | Building count per square kilometre within the 500m buffer |
| `building_data_missing` | `True` when no building feature was matched |
| `commercial_pct_500m` | Percentage of mapped land-use area classified as commercial |
| `green_recreation_pct_500m` | Percentage of mapped land-use area classified as green or recreation space |
| `industrial_pct_500m` | Percentage of mapped land-use area classified as industrial |
| `institutional_pct_500m` | Percentage of mapped land-use area classified as institutional |
| `residential_pct_500m` | Percentage of mapped land-use area classified as residential |
| `transport_other_pct_500m` | Percentage of mapped land-use area classified as transport or other land use |
| `landuse_data_missing` | `True` when all land-use percentage features are zero |

#### `POIs_output/sydney_station_urban_context_500m.gpkg`

GeoPackage version of the final dataset. It contains the same station-level features as the CSV output, plus station geometry.

## Environment

The notebooks were developed with Python 3.13. Core packages:

- pandas
- geopandas
- pyogrio
- numpy
- matplotlib
- osmnx

Install dependencies with:

```bash
pip install -r requirements.txt
```

GeoPandas and Pyogrio depend on geospatial system libraries. If installation with `pip` fails, a Conda environment is recommended:

```bash
conda create -n urban-context python=3.13 pandas geopandas pyogrio numpy matplotlib osmnx -c conda-forge
conda activate urban-context
```

## How to Reproduce

1. Place the Transport for NSW station-reference metadata in the project root:

   ```text
   road_traffic_counts_station_reference.csv
   ```

2. Place the Sydney OSM extract at:

   ```text
   data/osm/Sydney.osm.pbf
   ```

3. Run the station candidate selection notebook:

   ```text
   notebooks/01_station_candidate_selection.ipynb
   ```

   This creates:

   ```text
   sydney_permanent_stations_candidate_1.csv
   ```

4. Run the urban context feature extraction notebook:

   ```text
   notebooks/02_urban_context_feature_extraction.ipynb
   ```

5. Check the generated files in:

   ```text
   POIs_output/
   ```

## Notes and Limitations

- POI and land use coverage depends on OpenStreetMap completeness.
- Some stations may have zero matched POIs, buildings, or land use records. These cases are flagged with `*_data_missing` columns.
- The formal workflow reads from a local `.osm.pbf` extract to avoid repeated online OSM API requests.
- Large raw geospatial files should not usually be committed directly to GitHub unless Git LFS is configured.

## Suggested Citation

OpenStreetMap contributors. OpenStreetMap data for Sydney, accessed through a local `.osm.pbf` extract.
