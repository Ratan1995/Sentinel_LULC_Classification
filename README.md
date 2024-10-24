# Sentinel LULC Classification using Random Forest

## Description
This repository contains a supervised land use and land cover classification project using Google Earth Engine, Sentinel-1, and Sentinel-2 satellite data. The classification focuses on three distinct seasons (hot, wet, and dry) based on World Bank seasonal adaptation. A Random Forest classifier is used to classify the land cover into five categories (water, built-up, vegetation, road, and barren).

## Data Sources
- **Sentinel-1**: Radar imagery providing VV and VH polarizations.
- **Sentinel-2**: Optical imagery with NDVI, NDWI, and MSAVI indices.
- **SRTM DEM**: For slope data.

## Seasons
The classification is performed on three different seasonal composites:
- **Hot and dry season** (August to October)
- **Wet season** (January to April, and November to December)
- **Cool dry season** (May to July)

## Features
- **Sentinel-2 Indices**: NDVI, NDWI, MSAVI
- **Sentinel-1**: VV/VH band ratio
- **DEM**: Slope calculated from SRTM

## Workflow Overview
1. **Data Preprocessing**:
   - Cloud masking for Sentinel-2.
   - Slope calculation from SRTM DEM.
   - Seasonal composites for Sentinel-2 imagery.
2. **Random Forest Classification**:
   - A Random Forest model is trained using ground truth data from forest/non-forest samples.
   - Classification is performed on Sentinel-1 and Sentinel-2 composites.
3. **Accuracy Assessment**:
   - The model accuracy is evaluated using an error matrix based on test samples.

## Code
The script is written in JavaScript and runs in Google Earth Engine (GEE). To use it:
1. Set up an account on Google Earth Engine.
2. Create a new script and paste the code provided in `landcover_classification.js`.
3. Adjust the study area and sample data if needed.

