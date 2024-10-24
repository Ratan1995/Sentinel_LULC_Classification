// Supervised Classification using Sentinel 1 and 2

// 3 seasons: 
//  - Hot and dry season (mid-August to mid-November)
//  - Wet rainy season (mid-November to April)
//  - Cool dry season (May to mid-August)
// Source: World Bank (adapted)

// Adaptation: 
//  - Hot: August to October
//  - Wet: January to April / November and December
//  - Dry: May to July

// Study area: Define the geometry
var kat = ee.FeatureCollection('users/mpicoli/ZMB_KAT_buffer5km_');
var geometry = kat;
Map.centerObject(geometry);

//------------- Slope - SRTM 30 m -------------
// Calculate slope from SRTM DEM and reduce noise using a median filter
var srtm = ee.Image('USGS/SRTMGL1_003');
var slope = ee.Terrain.slope(srtm).clip(geometry);
var slope = slope.reduceNeighborhood({
  reducer: ee.Reducer.median(),
  kernel: ee.Kernel.circle(5),
}).rename('slope');

var addslope = function(image) {
  return image.addBands(slope).uint16();
};

//------------- Sentinel-2 Processing -------------
// Mask clouds and cirrus in Sentinel-2 images
function maskS2clouds(image) {
  var qa = image.select('QA60');
  var cloudBitMask = 1 << 10;
  var cirrusBitMask = 1 << 11;
  var mask = qa.bitwiseAnd(cloudBitMask).eq(0).and(qa.bitwiseAnd(cirrusBitMask).eq(0));
  return image.updateMask(mask).divide(10000);
}

//------------ Hot Season ------------
var S2_hot = ee.ImageCollection('COPERNICUS/S2_SR')
                  .filterDate('2022-08-01', '2022-10-31')
                  .filterBounds(geometry)
                  .filter(ee.Filter.lt('CLOUDY_PIXEL_PERCENTAGE', 20))
                  .map(maskS2clouds);

// Add NDVI, NDWI, and MSAVI for hot season
var addNDVI = function(image) {
  return image.addBands(image.normalizedDifference(['B8', 'B4']).rename('ndvi'));
};
var addNDWI = function(image) {
  return image.addBands(image.normalizedDifference(['B3', 'B8']).rename('ndwi'));
};
var MSAVI = function(image) {
  var msavi = image.expression(
    '(2 * NIR + 1 - sqrt(pow((2 * NIR + 1), 2) - 8 * (NIR - RED)) ) / 2', {
    'NIR': image.select('B8'),
    'RED': image.select('B4')
  }).rename('msavi');
  return image.addBands(msavi).float();
};

var S2_hot = S2_hot.map(addNDVI).map(addNDWI).map(MSAVI);
var compositionS2_hot = S2_hot.median().select(['B2', 'B3', 'B4', 'B8', 'ndvi', 'ndwi', 'msavi']);

//----------- Dry Season -----------
var S2_dry = ee.ImageCollection('COPERNICUS/S2_SR')
                  .filterDate('2022-05-01', '2022-07-31')
                  .filterBounds(geometry)
                  .filter(ee.Filter.lt('CLOUDY_PIXEL_PERCENTAGE', 20))
                  .map(maskS2clouds)
                  .map(addNDVI)
                  .map(addNDWI)
                  .map(MSAVI);

var compositionS2_dry = S2_dry.median().select(['B2', 'B3', 'B4', 'B8', 'ndvi', 'ndwi', 'msavi']);

//----------- Wet Season -----------
var S2_wet = ee.ImageCollection('COPERNICUS/S2_SR')
                  .filter(ee.Filter.or(
                    ee.Filter.date('2022-01-01', '2022-04-30'),
                    ee.Filter.date('2022-11-01', '2022-12-31')))
                  .filterBounds(geometry)
                  .filter(ee.Filter.lt('CLOUDY_PIXEL_PERCENTAGE', 20))
                  .map(maskS2clouds)
                  .map(addNDVI)
                  .map(addNDWI)
                  .map(MSAVI);

var compositionS2_wet = S2_wet.median().select(['B2', 'B3', 'B4', 'B8', 'ndvi', 'ndwi', 'msavi']);

// Combine all seasons and slope into a final composition
var compositionS2 = compositionS2_hot.addBands(compositionS2_dry).addBands(compositionS2_wet).addBands(slope);
print(compositionS2, 'composition S2');

//------------- Sentinel-1 Processing -------------
var S1 = ee.ImageCollection('COPERNICUS/S1_GRD')
            .filterDate('2022-05-01', '2022-06-30')
            .filterBounds(geometry)
            .select(['VV', 'VH']);

// Calculate the VV/VH ratio and add to bands
var ratio = function(image) {
  return image.addBands(image.select('VV').divide(image.select('VH')).rename('vv_vh'));
};
var S1 = S1.map(ratio);
var compositionS1 = S1.median().float();

// Combine Sentinel-1 and Sentinel-2 composites
var composite = ee.Image.cat([compositionS1, compositionS2]);
print(composite, 'composite S1 and S2');
Map.addLayer(composite, {}, 'composite S1 and S2', false);

//------------- Random Forest (RF) Model -------------
// Load training data
var samples_kat = ee.FeatureCollection('users/mpicoli/KAT_forest_nonforest_2018_2021_v6');
var label = 'new_label';

var trainingImage = composite.sampleRegions({
  collection: samples_kat,
  properties: [label],
  geometries: true,
  scale: 10
});

// Split data into training and testing sets
var training = trainingImage.randomColumn();
var trainSet = training.filter(ee.Filter.lessThan('random', 0.8));
var testSet = training.filter(ee.Filter.greaterThanOrEquals('random', 0.8));
print(trainSet, "trainSet");
print(testSet, "testSet");

// Train Random Forest model
var classifier = ee.Classifier.smileRandomForest(500).train({
  features: trainSet,
  classProperty: label,
  inputProperties: composite.bandNames()
});

// Classify the input imagery
var classified = composite.classify(classifier);

// Define color palette for land cover classes
var landcoverPalette = [
  '253494', // Water
  '969696', // Buildup
  'FF8000', // Vegetation
  '006837', // Road
  '000000', // Barren
];

Map.addLayer(classified, {palette: landcoverPalette, min: 1, max: 6}, 'classification 2022', false);

// Accuracy assessment
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
