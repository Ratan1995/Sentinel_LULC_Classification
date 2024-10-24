# Sentinel_LULC_Classification
Supervised land cover classification using Sentinel-1 and Sentinel-2 with Random Forest in Earth Engine.
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

### Example Usage
//Supervised Classification using Sentinel 1 and 2

// 3 seasons: a hot and dry season (mid-August to mid-November),
//           a wet rainy season (mid-November to April) 
//           a cool dry season (May to mid-August). Source: World Bank
// Adptation: hot (August to October), wet (Jan to April / Nov and Dec), dry (May to July)

//Study area
var kat = ee.FeatureCollection('users/mpicoli/ZMB_KAT_buffer5km_');
var geometry = kat;
Map.centerObject(geometry);

//-------------Slope - SRTM 30 m-------------
var srtm = ee.Image('USGS/SRTMGL1_003');
var slope = ee.Terrain.slope(srtm).clip(geometry);
// DEM was degraded with a 5 x 5 pixel median filter before calculating slope to reduce noise.
var slope = slope.clip(geometry).reduceNeighborhood({
  reducer: ee.Reducer.median(),
  kernel: ee.Kernel.circle(5),
}).rename('slope');
var addslope = function(image) {
  return image
  .addBands(slope).uint16();
};


//-------------Sentinel 2----------------
//Mask cloud
function maskS2clouds(image) {
  var qa = image.select('QA60');
  // Bits 10 and 11 are clouds and cirrus, respectively.
  var cloudBitMask = 1 << 10;
  var cirrusBitMask = 1 << 11;
  // Both flags should be set to zero, indicating clear conditions.
  var mask = qa.bitwiseAnd(cloudBitMask).eq(0)
      .and(qa.bitwiseAnd(cirrusBitMask).eq(0));
  return image.updateMask(mask).divide(10000);
}

//------------Hot season------------
var S2_hot = ee.ImageCollection('COPERNICUS/S2_SR')
                  .filterDate('2022-08-01', '2022-10-31')
                  .filterBounds(geometry)
                  .filter(ee.Filter.lt('CLOUDY_PIXEL_PERCENTAGE',20))
                  .map(maskS2clouds);               
//NDVI 
var addNDVI = function(image) {
 return image.addBands(image.normalizedDifference(['B8', 'B4']).rename('ndvi'));};
var S2_hot = S2_hot.map(addNDVI);
// NDWI 
var addNDWI = function(image) {
 return image.addBands(image.normalizedDifference(['B3', 'B8']).rename('ndwi'));};
var S2_hot = S2_hot.map(addNDWI);
//MSAVI
var MSAVI = function (image) {
    var msavi = image.expression(
    '(2 * NIR + 1 - sqrt(pow((2 * NIR + 1), 2) - 8 * (NIR - RED)) ) / 2', {
      'NIR': image.select('B8'), 'RED': image.select('B4'),}).rename('msavi');
    return image.addBands(msavi).float();};
var S2_hot = S2_hot.map(MSAVI);
var compositionS2_hot = S2_hot.median().select(['B2', 'B3', 'B4', 'B8','ndvi','ndwi', 'msavi']);

//-----------Dry season--------
var S2_dry = ee.ImageCollection('COPERNICUS/S2_SR')
                  .filterDate('2022-05-01', '2022-07-31')
                  .filterBounds(geometry)
                  .filter(ee.Filter.lt('CLOUDY_PIXEL_PERCENTAGE',20))
                  .map(maskS2clouds);               
//NDVI 
var S2_dry = S2_dry.map(addNDVI);
// NDWI 
var S2_dry = S2_dry.map(addNDWI);
//MSAVI
var S2_dry = S2_dry.map(MSAVI);
var compositionS2_dry = S2_dry.median().select(['B2', 'B3', 'B4', 'B8','ndvi','ndwi', 'msavi']);

//-----------Wet season--------
var S2_wet = ee.ImageCollection('COPERNICUS/S2_SR')
                    .filter(ee.Filter.or(
                     ee.Filter.date('2022-01-01', '2022-04-30'),
                     ee.Filter.date('2022-11-01', '2022-12-31')))
                    .filterBounds(geometry)
                    .filter(ee.Filter.lt('CLOUDY_PIXEL_PERCENTAGE',20))
                    .map(maskS2clouds);               
//NDVI 
var S2_wet = S2_wet.map(addNDVI);
// NDWI 
var S2_wet = S2_wet.map(addNDWI);
//MSAVI
var S2_wet = S2_wet.map(MSAVI);
var compositionS2_wet = S2_wet.median().select(['B2', 'B3', 'B4', 'B8','ndvi','ndwi', 'msavi']);

//final composition
var compositionS2 = compositionS2_hot.addBands(compositionS2_dry).addBands(compositionS2_wet).addBands(slope);
print (compositionS2, 'composition S2');

//------------------ Sentinel-1------------------ 
var S1 = ee.ImageCollection('COPERNICUS/S1_GRD')
          .filterDate('2022-05-01', '2022-06-30')
          .filterBounds(geometry)
          .select(['VV', 'VH']);

// Calculate the band ratio of VV and VH from Sentinel-1 image
var ratio = function(image){
  return image.addBands(image.select('VV').divide(image.select('VH')).rename('vv_vh'));
};

var S1 = S1.map(ratio);
var compositionS1 = S1.median();
var compositionS1 = compositionS1.float();

//------------------ composite Sentinel-2 and Sentinel-1-------------- 
var composite = ee.Image.cat([compositionS1, compositionS2]);
print(composite, 'composite S1 and S2');
Map.addLayer(composite, {}, 'composite S1 and S2', false);


//---------------------RF model --------------------
//samples
var samples_cop = ee.FeatureCollection('users/mpicoli/COP_forest_nonforest_2015_2021');
var samples_kat = ee.FeatureCollection('users/mpicoli/KAT_forest_nonforest_2018_2021_v6');
var trainingSamples = samples_kat;
var label = 'new_label'; 

var trainingImage = composite.sampleRegions({
  collection: trainingSamples,
  properties: [label],
  geometries: true,
  scale: 10
});

var training = trainingImage.randomColumn();
var trainSet = training.filter(ee.Filter.lessThan('random', 0.8));
var testSet = training.filter(ee.Filter.greaterThanOrEquals('random', 0.8));
print(trainSet,"trainSet");
print(testSet, "testSet");
// Classification Model
var classifier = ee.Classifier
                 .smileRandomForest(500)
                 .train({
                   features: trainSet, 
                   classProperty: label, 
                   inputProperties: composite.bandNames()});

//Classify the input imagery.
var classified = composite.classify(classifier);

// Define a palette for the classification.
var landcoverPalette = [
  '253494', //Water (0)
  '969696', //Buildup (1)
  'FF8000', //Vegetation (2)
  '006837', //Road (3)
  '000000', //Barren (4)
];

Map.addLayer(classified, {palette: landcoverPalette, min: 1, max: 6}, 'classification 2022', false);

// Accuracy Assessment
    
var tested = testSet.classify(classifier, 'classification');                  
var testAccuracy = tested.errorMatrix('new_label', 'classification');
print(testAccuracy, 'Test samples - error matrix');
print(testAccuracy.accuracy(), 'Test samples - accuracy');

// Export classified map to Google Drive
Export.image.toDrive({
  image: classified,
  description: 'Sentinel_LULC_2022',
  scale: 10,
  region: geometry,
  fileFormat: 'GeoTIFF',
  folder: 'RF_classification',
  maxPixels: 1e13,
});
