# Eyob-Repository
Flood hazard mapping and Assessment Using GEE
Imports (4 entries)
var hydrosheds = ee.Image("WWF/HydroSHEDS/03VFDEM"),
    gsw = ee.Image("JRC/GSW1_4/GlobalSurfaceWater"),
    polygon = ee.FeatureCollection("users/eyobdangiso94/Perimetere"),   // The study area shapefile
    s1 = ee.ImageCollection("COPERNICUS/S1_GRD");

    // Function to convert from dB to linear scale
function toNatural(img) {
  // Convert from dB to linear scale
  return ee.Image(10.0).pow(img.select(0).divide(10.0));
}

// Function to convert to dB
function toDB(img) {
  return ee.Image(img).log10().multiply(10.0);
}

// Create a feature from the polygon geometry
var feature = ee.Feature(polygon, {
  name: 'Study Area'
});

// Print the created feature
print('Created Feature:', feature);

// Add the polygon to the map
Map.addLayer(polygon, {color: 'blue'}, 'Study Area');
Map.centerObject(polygon, 8);

// Select images by predefined dates
var beforeStart = '2020-07-28';
var beforeEnd = '2020-08-05';
var afterStart = '2020-08-05';
var afterEnd = '2020-09-10';

// Assuming s1 is defined correctly as a Sentinel-1 image collection
var filtered = s1
  .filter(ee.Filter.eq('instrumentMode', 'IW'))
  .filter(ee.Filter.listContains('transmitterReceiverPolarisation', 'VH'))
  .filter(ee.Filter.listContains('transmitterReceiverPolarisation', 'VV'))
  .filter(ee.Filter.eq('orbitProperties_pass', 'DESCENDING'))
  .filter(ee.Filter.eq('resolution_meters', 10))
  .filterBounds(polygon)
  .select(['VV', 'VH']);

var before = filtered.filterDate(beforeStart, beforeEnd).mosaic().clip(polygon);
var after = filtered.filterDate(afterStart, afterEnd).mosaic().clip(polygon);

var addRatioBand = function(image) {
  var ratioBand = image.select('VV').divide(image.select('VH')).rename('VV/VH');
  return image.addBands(ratioBand);
};

var beforeRgb = addRatioBand(before);
var afterRgb = addRatioBand(after);
print(beforeRgb);

print(afterRgb);

print(filtered.first());

var visParams = {min:[-25, -25, 0] ,max:[0, 0, 2]};
Map.centerObject(polygon, 10);
Map.addLayer(beforeRgb, visParams, 'Before Floods');
Map.addLayer(afterRgb, visParams, 'After Floods');

var beforeFiltered = ee.Image(toDB(toNatural(before)));
var afterFiltered = ee.Image(toDB(toNatural(after)));
Map.addLayer(beforeFiltered, {min:-25,max:0}, 'Before Filtered', false);
Map.addLayer(afterFiltered, {min:-25,max:0}, 'After Filtered', false);

var difference = afterFiltered.divide(beforeFiltered);

// Define a threshold
var diffThreshold = 1.25;

// Initial estimate of flooded pixels
var flooded = difference.gt(diffThreshold).rename('water').selfMask();
Map.addLayer(flooded, {min:0, max:1, palette:['orange']}, 'Initial Flood Area',false);

// Mask out area with permanent/semi-permanent water
var permanentWater = gsw.select('seasonality').gte(5).clip(polygon)
Map.addLayer(permanentWater.selfMask(), {min:0, max:1, palette: ['blue']}, 'Permanent Water', false)

// GSW data is masked in non-water areas. Set it to 0 using unmask()
// Invert the image to set all non-permanent regions to 1
var permanentWaterMask = permanentWater.unmask(0).not()

var flooded = flooded.updateMask(permanentWaterMask)
// Mask out areas with more than 5 percent slope using the HydroSHEDS DEM
var slopeThreshold = 5;
var terrain = ee.Algorithms.Terrain(hydrosheds);
var slope = terrain.select('slope');
var steepAreas = slope.gt(slopeThreshold)
var slopeMask = steepAreas.not()
Map.addLayer(steepAreas.selfMask(), {min:0, max:1, palette: ['cyan']}, 'Steep Areas', false)

var flooded = flooded.updateMask(slopeMask)

// Remove isolated pixels
// connectedPixelCount is Zoom dependent, so visual result will vary
var connectedPixelThreshold = 8;
var connections = flooded.connectedPixelCount(25)
var disconnectedAreas = connections.lt(connectedPixelThreshold)
var disconnectedAreasMask = disconnectedAreas.not()
Map.addLayer(disconnectedAreas.selfMask(), {min:0, max:1, palette: ['yellow']}, 'Disconnected Areas', false)

var flooded = flooded.updateMask(disconnectedAreasMask)

Map.addLayer(flooded, {min:0, max:1, palette: ['red']}, 'Flooded Areas');

// Calculate the total area of all features in the polygon collection
var totalArea = polygon.geometry().area().divide(10000); // Convert to hectares
print('Total Area (ha):', totalArea);

var stats = flooded.multiply(ee.Image.pixelArea()).reduceRegion({
  reducer: ee.Reducer.sum(),
  geometry: polygon,
  scale: 10,
  maxPixels: 1e10,
  tileScale: 16
})
print(stats)


var floodedArea = ee.Number(stats.get('water')).divide(10000)
print(floodedArea)


// Create a FeatureCollection containing the single feature
//var featureCollection = ee.FeatureCollection([feature]);

// Create a FeatureCollection containing the single feature
//var featureCollection = ee.FeatureCollection([feature]);

// Convert the flooded raster image to a binary raster
var floodedAreaBinary = flooded.gt(0);

// Export the flooded area as a shapefile
Export.table.toDrive({
  collection: floodedAreaBinary.reduceToVectors({
    geometry: polygon,
    scale: 25,
    maxPixels: 1e9, // Adjust the maxPixels value
    eightConnected: false,
    labelProperty: 'water'
  }),
  description: 'flooded_area_shapefile',
  folder: 'GEE_exports',
  fileFormat: 'SHP'
});

// Export Disconnected Areas
Export.image.toDrive({
  image: disconnectedAreasMask,
  description: 'Disconnected_Areas',
  folder: 'GEE_exports',
  region: polygon.geometry(),
  scale: 10,
  maxPixels: 1e13,
  fileFormat: 'GEOTIFF',
  formatOptions: {
    cloudOptimized: true
  }
});

// Convert the flooded raster image to a binary raster
var permanentWaterBinary = flooded.gt(0);

// Export the permanent Water as a shapefile
Export.table.toDrive({
  collection: permanentWaterBinary.reduceToVectors({
    geometry: polygon,
    scale: 20,
    maxPixels: 1e9, // Adjust the maxPixels value
    eightConnected: false,
    labelProperty: 'water'
  }),
  description: 'permanent_Water_shapefile',
  folder: 'GEE_exports',
  fileFormat: 'SHP'
});

// Convert the Initial Flood Area raster image to a binary raster
var InitialFloodAreaBinary = flooded.gt(0);

// Export the Initial flooded area as a shapefile
Export.table.toDrive({
  collection: InitialFloodAreaBinary.reduceToVectors({
    geometry: polygon,
    scale: 20,
    maxPixels: 1e9, // Adjust the maxPixels value
    eightConnected: false,
    labelProperty: 'water'
  }),
  description: 'Initial_Flood_Area_shapefile',
  folder: 'GEE_exports',
  fileFormat: 'SHP'
});

// Export After Filtered
Export.image.toDrive({
  image: afterFiltered,
  description: 'After_Filtered',
  folder: 'GEE_exports',
  region: polygon.geometry(),
  scale: 10,
  maxPixels: 1e13,
  fileFormat: 'GeoTIFF',
  formatOptions: {
    cloudOptimized: true
  }
});

// Export Before Filtered
Export.image.toDrive({
  image: beforeFiltered,
  description: 'Before_Filtered',
  folder: 'GEE_exports',
  region: polygon.geometry(),
  scale: 10,
  maxPixels: 1e13,
  fileFormat: 'GeoTIFF',
  formatOptions: {
    cloudOptimized: true
  }
});

// Export After Floods
Export.image.toDrive({
  image: afterRgb,
  description: 'After_Floods',
  folder: 'GEE_exports',
  region: polygon.geometry(),
  scale: 10,
  maxPixels: 1e13,
  fileFormat: 'GeoTIFF',
  formatOptions: {
    cloudOptimized: true
  }
});

// Export Before Floods
Export.image.toDrive({
  image: beforeRgb,
  description: 'Before_Floods',
  folder: 'GEE_exports',
  region: polygon.geometry(),
  scale: 10,
  maxPixels: 1e13,
  fileFormat: 'GeoTIFF',
  formatOptions: {
    cloudOptimized: true
  }
});
