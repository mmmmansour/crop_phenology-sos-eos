# crop_phenology-sos-eos



// Définir une région d'intérêt (ROI)
// var roi = ee.Geometry.Polygon([
//   [[-1.5, 48.0], [-1.5, 48.5], [-1.0, 48.5], [-1.0, 48.0], [-1.5, 48.0]]
// ]); // Exemple : une zone près de la Bretagne, France

// Importer la collection Sentinel-1
var sentinel1 = ee.ImageCollection('COPERNICUS/S1_GRD')
  .filter(ee.Filter.listContains('transmitterReceiverPolarisation', 'VV'))
  .filter(ee.Filter.listContains('transmitterReceiverPolarisation', 'VH'))
  .filter(ee.Filter.eq('instrumentMode', 'IW'))
  .filterBounds(roi) // Filtrer par la région d'intérêt
  .filterDate('2020-01-01', '2024-12-01'); // Ajustez les dates selon vos besoins

// Fonction pour calculer l'humidité du sol pour une image
function calculateSoilMoisture(image) {
  var vv = image.select('VV');

  // Calcul des statistiques sur la région
  var stats = vv.reduceRegion({
    reducer: ee.Reducer.minMax().combine({
      reducer2: ee.Reducer.mean(),
      sharedInputs: true
    }),
    geometry: roi,
    scale: 10,
    maxPixels: 1e9
  });

  var maxVV = ee.Number(stats.get('VV_max'));
  var minVV = ee.Number(stats.get('VV_min'));

  // Sensibilité (intervalle d'intensité)
  var sensitivity = maxVV.subtract(minVV);

  // Calcul de l'indice d'humidité (Mv)
  var Mv = vv.subtract(minVV).divide(sensitivity);

  // Conversion en millimètres d'humidité
  var soilDepth = 50; // Profondeur effective du sol (mm, ici 5 cm)
  var maxVSM = 0.4; // Capacité de rétention maximale du sol (fraction volumique, 40%)
  var soilMoisture_mm = Mv.multiply(maxVSM).multiply(soilDepth);

  return soilMoisture_mm.clip(roi); // Retourne l'image d'humidité pour cette ROI
}

// Appliquer le calcul à toutes les images
var soilMoistureCollection = sentinel1.map(calculateSoilMoisture);

// Calculer la moyenne temporelle de l'humidité
var meanSoilMoistureImage = soilMoistureCollection.mean();

// Ajouter la couche moyenne d'humidité sur la carte
var visParams = {
  min: 0,
  max: 20, // Ajustez selon les conditions locales
  palette: ['white', 'blue', 'green', 'yellow', 'red']
};
var palettes = require('users/gena/packages:palettes');

// Choose and define a palette
var palette = palettes.crameri.roma[10];
Map.centerObject(roi, 10); // Centrer la carte sur la ROI
Map.addLayer(meanSoilMoistureImage.focal_median(20,'circle','meters')
, {palette:palette}, 'Humidité moyenne (mm)');

// Ajouter la ROI sur la carte (facultatif)
Map.addLayer(roi, {color: 'red'}, 'ROI');



var viz = {palette: palette, min: 8, max: 16};

var legend = ui.Panel({
  style: {
    position: 'bottom-left',
    padding: '8px 15px'
  }
});

// Create and add the legend title.
var legendTitle = ui.Label({
  value: 'humidity (mm)',
  style: {
    fontWeight: 'bold',
    fontSize: '18px',
    margin: '0 0 4px 0',
    padding: '0'
  }
});

legend.add(legendTitle);

// create the legend image
var lon = ee.Image.pixelLonLat().select('latitude');
var gradient = lon.multiply((viz.max-viz.min)/100.0).add(viz.min);
var legendImage = gradient.visualize(viz);
 
// create text on top of legend
var panel = ui.Panel({
widgets: [
ui.Label(viz['max'])
],
});
 
legend.add(panel);
 
// create thumbnail from the image
var thumbnail = ui.Thumbnail({
image: legendImage,
params: {bbox:'0,0,10,100', dimensions:'10x200'},
style: {padding: '1px', position: 'bottom-center'}
});
 
// add the thumbnail to the legend
legend.add(thumbnail);
 
// create text on top of legend
var panel = ui.Panel({
widgets: [
ui.Label(viz['min'])
],
});

legend.add(panel);
Map.add(legend);
