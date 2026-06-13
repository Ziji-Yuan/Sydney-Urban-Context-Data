# Sydney Urban Context Data

Urban context feature extraction for Sydney traffic stations using OpenStreetMap (OSM), POIs, and land use data.

## Overview

This project focuses on constructing station-level urban context features for Sydney traffic monitoring stations.

The current urban context components include:

- Points of Interest (POIs)
- Land Use

The extracted features are designed to support future spatio-temporal reasoning and benchmark tasks.

## Study Area

Sydney Metropolitan Area, NSW, Australia.

Traffic stations are used as spatial anchors for urban context feature extraction.

## Data Sources

### Traffic Stations

- NSW Traffic Count Data
- Permanent Sydney Traffic Monitoring Stations

### Urban Context Data

- OpenStreetMap (OSM)
- POI datasets
- Land use datasets

## Workflow

1. Prepare traffic station coordinates.
2. Generate station buffers.
3. Extract nearby POIs.
4. Extract land use features.
5. Generate station-level urban context features.

## Repository Structure

```text
Sydney-Urban-Context-Data/

├── data/
├── notebooks/
├── outputs/
└── README.md
```

## Expected Output

Final output example:

| station_key | school_count | hospital_count | restaurant_count | residential_ratio |
|------------|--------------|----------------|------------------|-------------------|
| 57052 | 4 | 1 | 27 | 0.65 |

Each row represents urban context characteristics surrounding a traffic monitoring station.

## Author

Ziji Yuan  
UNSW Sydney
