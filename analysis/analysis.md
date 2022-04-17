**Hospital site suitability analysis using JavaScript in Google Earth
Engine (GEE)**

Adrienne Hall, April 2022

Health services and facilities are social investments. Geospatial
analysis is useful for making decisions about where to invest in health
resources. This sample project uses **suitability analysis** to identify
possible locations to site a public hospital in North Carolina. A
suitability index is a "system whereby locations are ranked according to
how well they fit a set of criteria" (Shellito, 2020, pg. 200).
Suitability analysis seeks to answer the question: where is the best
option for something to be located?

Six factors of relevance will be used to create a simple suitability
index and rank ideal locations.

1.  Nearby existing medical facilities (derived from state health
    facilities)

2.  High population density (input layer from GEE)

3.  Proximate to public water (derived from public water supply sources
    layer)

4.  Proximate to public sewage system (derived from public sewage
    systems layer)

5.  Proximity to public roads (derived from state-maintained roads
    layer)

6.  Accessible slope degree (derived from elevation dataset from GEE)

**Suitability Index**

| Input                                             | Range              | Score (1-worst to 5-best) |     |
| ------------------------------------------------- | ------------------ | ------------------------- | --- |
| Distance to existing medical facilities           |                    |                           |     |
|                                                   | \> 16 km           | 5                         |     |
|                                                   | 12 to 16 km        | 4                         |     |
|                                                   | 8 to 12 km         | 3                         |     |
|                                                   | 4 to 8 km          | 2                         |     |
|                                                   | 0 to 4 km          | 1                         |     |
| Population density (persons per square kilometer) |                    |                           |     |
|                                                   | \> 500             | 5                         |     |
|                                                   | 250 to 500         | 4                         |     |
|                                                   | 100 to 250         | 3                         |     |
|                                                   | 50 to 100          | 2                         |     |
|                                                   | 0 to 50            | 1                         |     |
| Distance to public water                          |                    |                           |     |
|                                                   | 0 to 100 meters    | 5                         |     |
|                                                   | 100 to 250 meters  | 4                         |     |
|                                                   | 250 to 500 meters  | 3                         |     |
|                                                   | 500 to 1000 meters | 2                         |     |
|                                                   | \> 1000 meters     | 1                         |     |
| Distance to public sewage                         |                    |                           |     |
|                                                   | 0 to 100 meters    | 5                         |     |
|                                                   | 100 to 250 meters  | 4                         |     |
|                                                   | 250 to 500 meters  | 3                         |     |
|                                                   | 500 to 1000 meters | 2                         |     |
|                                                   | \> 1000 meters     | 1                         |     |
| Distance to public road                           |                    |                           |     |
|                                                   | 0 to 500 meters    | 5                         |     |
|                                                   | 500 to 2000 meters | 3                         |     |
|                                                   | \> 2000 meters     | 1                         |     |
| Slope degree                                      |                    |                           |     |
|                                                   | 0 to 1             | 5                         |     |
|                                                   | 1 to 2             | 4                         |     |
|                                                   | 2 to 3             | 3                         |     |
|                                                   | 4 to 5             | 2                         |     |
|                                                   | \> 5               | 1                         |     |
|                                                   |                    |                           |     |

**GEE Code:**

`````javascript
// Import SRTM dataset, select elevation band, create elevation and slope variables
var dataset = ee.Image('NASA/NASADEM_HGT/001');
var elevation = dataset.select('elevation'); // select the elevation band
//derive slope from SRTM elevation dataset
var slope = ee.Terrain.slope(elevation);

// Import UN World Population density dataset, set visualization properties, and clip to NC
var dataset = ee.ImageCollection("CIESIN/GPWv411/GPW_UNWPP-Adjusted_Population_Density").first();
var popDEN = dataset.select('unwpp-adjusted_population_density');// select band
var popDEN_vis = {
  "max": 1000.0,
  "palette": [
    "ffffe7",
    "FFc869",
    "ffac1d",
    "e17735",
    "f2552c",
    "9f0c21"
  ],
  "min": 0.0
};

//create distance rasters from inputs and store as variables
var medDIS = medical.distance().clip(nc);
var waterDIS = water.distance().clip(nc);
var sewerDIS = sewer.distance().clip(nc);
var roadDIS = road.distance().clip(nc);

//reclassify layers using an expression
var medRC = medDIS.expression('b(0) <= 16000 ? 5 : b(0) <= 12000 ? 4 : b(0) <= 8000 ? 3 : b(0) <= 4000 ? 2 : 1');
var popRC = popDEN.expression('b(0) <= 500 ? 5 : b(0) <= 250 ? 4 : b(0) <= 100 ? 3 : b(0) <= 50 ? 2 : 1');
var waterRC = waterDIS.expression('b(0) <= 100 ? 5 : b(0) <= 250 ? 4 : b(0) <= 500 ? 3 : b(0) <= 1000 ? 2 : 1');
var sewerRC = sewerDIS.expression('b(0) <= 100 ? 5 : b(0) <= 250 ? 4 : b(0) <= 500 ? 3 : b(0) <= 1000 ? 2 : 1');
var roadRC = roadDIS.expression('b(0) <= 500 ? 5 : b(0) <= 2000 ? 3 : 1');
var slopeRC = slope.expression('b(0) <= 1 ? 5 : b(0) <= 2 ? 4 : b(0) <= 3 ? 3 : b(0) <= 4 ? 2 : 1');

// create suitability output layer
var suitability = medRC.add(popRC).add(waterRC).add(sewerRC).add(roadRC).add(slopeRC);
// //create color palette from color brewer
var spectral = ['#a50026','#d73027','#f46d43','#fdae61','#fee090','#ffffbf','#e0f3f8','#abd9e9','#74add1','#4575b4','#313695'];

// Add a white background image to the map.
var background = ee.Image(1);
Map.addLayer(background, {min: 0, max: 1}, 'Background');

// add suitability output layer to map.
Map.addLayer(suitability.clip(nc), {min: 0, max: 30, palette: spectral}, 'Hospital Site Suitability Map');
Map.setCenter(-79.4968, 35.5806, 7); //set map center and zoom to NC (region of interest)

// Export the image, specifying the CRS, transform, and region.
Export.image.toDrive({
   image: suitability,
   description: 'SuitabilityMap',
   scale: 1000,
   region: geometry,
 });
`````

Repo: https://code.earthengine.google.com/?accept_repo=users/adriennehall8/GISUNC 


**Result:**

Suitability Map (Output Raster Styled in QGIS)

![Diagram, map Description automatically
generated](HospitalSiteSuitabilityMap.png)

[Download result in raster
format.](https://adriennehall8.github.io/portfolio/analysis/SuitabilityMap.tif)


**References**

Almansi, K. Y., Shariff, A. R. M., Abdullah, A. F., & Syed Ismail, S. N.
(2021). Hospital Site Suitability Assessment Using Three Machine
Learning Approaches: Evidence from the Gaza Strip in Palestine. *Applied
Sciences, 11*(22), 11054.

Parvin, F., Ali, S. A., Hashmi, S., & Khatoon, A. (2021). Accessibility
and site suitability for healthcare services using GIS-based hybrid
decision-making approach: a study in Murshidabad, India. *Spatial
Information Research, 29*(1), 1--18.
https://doi.org/10.1007/s41324-020-00330-0

Shellito, B. A. (2020). *Introduction to geospatial technologies*, Fifth
Edition. Macmillan Higher Education.
