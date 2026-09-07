# Nyabarongo Enhanced Flood Susceptibility Model

## Project overview

This repository documents the extension of an existing flood-susceptibility machine-learning model for the Nyabarongo catchment in Rwanda.

The original baseline model used a set of physical and environmental flood-conditioning variables. This work extended that model by deriving three additional variables from satellite and land-cover data:

- **NDVI_Change** – change in vegetation greenness between the 2018–2019 baseline and 2023.
- **BuiltPct** – percentage of each 200 m × 200 m grid cell classified as built-up in 2023.
- **WetDeclPct** – percentage of each 200 m × 200 m grid cell that showed significant wetness decline between the 2018–2019 baseline and 2023.

The purpose of these additions was to test whether indicators of vegetation change, built-up development, and drying of previously wetter areas improved flood-susceptibility prediction.

The work was divided into four main stages:

1. preparation of the study-area GIS data;
2. derivation and validation of the new Earth-observation variables;
3. integration of the new variables into the machine-learning datasets;
4. comparison of several machine-learning experiments.

---

# Repository structure

The repository was organised so that each major processing stage had its own folder.

```text
00_Original/
01_Boundary/
02_Sentinel2/
03_NDVI/
04_ISP_Built/
05_Wetlands/
06_QGIS_Output/
07_ML_Data/
08_Model_Output/
09_Validation/
```

The folders were used as follows:
```
00_Original/ contains the original study-area and sample-point datasets used by the baseline model.
01_Boundary/ contains the dissolved Nyabarongo study-area boundary created in QGIS.
02_Sentinel2/ contains files and scripts associated with Sentinel-2 image preparation.
03_NDVI/ contains NDVI and NDVI-change outputs.
04_ISP_Built/ contains the built-up analysis. Although the folder retained the earlier ISP name, the final variable was BuiltPct, not impervious surface percentage.
05_Wetlands/ contains the MNDWI and wetness-decline analysis.
06_QGIS_Output/ contains the processed 200 m grid outputs created in QGIS.
07_ML_Data/ contains the final training and prediction datasets used by the machine-learning experiments.
08_Model_Output/ contains model predictions and experiment outputs.
09_Validation/ contains validation material and quality-control outputs.
```
The QGIS project was stored as:

```text
Nyabarongo_Enhanced_Model.qgz
```
# Key terms

A small number of GIS and Earth-observation concepts were used throughout the project.

**Raster**:
A spatial dataset made from regularly spaced pixels. Satellite images such as Sentinel-2 are raster datasets.

**Vector layer**:
A spatial dataset represented by points, lines, or polygons. The flood sample locations were point features, while the 200 m study-area grid was made from polygon features.

**CRS – Coordinate Reference System**:
The system used to describe where spatial data were located on the Earth. CRS consistency was checked before spatial processing.

**Zonal statistics**:
A GIS operation that summarised raster values inside a polygon. In this project, raster pixels were summarised inside each 200 m × 200 m grid cell.


---

# 1. Initial project setup

A new project structure was created on GitHub so that the data-processing workflow, experiments, and outputs could be tracked in a reproducible way.

QGIS was installed and used for spatial-data inspection, validation, aggregation, and joining.

The original datasets from the baseline experiment were then loaded into QGIS:

 - the Nyabarongo study-area grid;
 - the original flood/non-flood sample points.

The study-area grid contained 213,908 polygon cells, each representing an approximately 200 m × 200 m analysis unit.

The original sample-point dataset contained 302 flood and non-flood observations.

The Coordinate Reference System of both layers was checked and recorded before any further processing was performed.

# 2. Creation of the Nyabarongo study-area boundary

The original study area consisted of 213,908 individual grid polygons.

Processing satellite imagery separately against this full grid would have been unnecessarily complex. A single study-area boundary was therefore created.

In QGIS, the Dissolve tool was used to merge all grid cells into one catchment boundary.

The dissolved boundary was then exported and stored in:

```01_Boundary/```

This boundary was subsequently used to restrict satellite processing to the Nyabarongo study area.

This reduced unnecessary processing outside the catchment and ensured that exported rasters corresponded to the modelling extent.

# 3. Google Earth Engine setup

Google Earth Engine (GEE) was used to process satellite imagery.

GEE was selected because it provided:

- free access to large Earth-observation datasets;
- direct access to Sentinel-2 imagery;
- cloud-based processing;
- the ability to process multi-year imagery without downloading every individual satellite scene.

The dissolved Nyabarongo boundary was uploaded to the GEE Assets area as a shapefile and imported into the processing scripts.

For example:

```var boundary = table;```

The boundary was then used to spatially filter and clip the satellite imagery.

# 4. Sentinel-2 time periods

Sentinel-2 Surface Reflectance imagery was used for the vegetation and wetness analyses.

The following periods were used:

- 1 July–31 December 2018
- 1 July–31 December 2019
- 1 July–31 December 2023

The same months were used in each year so that seasonal differences were reduced.

A direct comparison between different seasons could otherwise have produced apparent changes that were caused by normal seasonal vegetation or moisture cycles rather than genuine land-surface change.

The 2018 data contained substantial cloud contamination, so the 2018 and 2019 observations were combined to create a more robust baseline.

Clouds, cloud shadows, cirrus, and snow/ice were masked before the imagery was used.

# 5. NDVI change
## 5.1 Purpose

NDVI was used to measure vegetation greenness.

The Normalized Difference Vegetation Index was calculated as:

$$ NDVI = \frac{NIR - Red}{NIR + Red} $$

For Sentinel-2:

$$ NDVI = \frac{B8 - B4}{B8 + B4} $$

where:

- B8 represented near-infrared reflectance;
- B4 represented red reflectance.

Higher NDVI values generally indicated greener or denser vegetation.

## 5.2 NDVI-first processing method

NDVI was calculated for each cloud-masked Sentinel-2 image before annual composites were created.

This was referred to as the NDVI-first method.

For each year:

$$ NDVI_{year} = median(NDVI_1, NDVI_2, ..., NDVI_n) $$

A baseline was then created from the 2018 and 2019 annual composites.

Where both years contained valid data, both years contributed to the baseline. Where only one baseline year contained valid observations, the valid year was retained rather than creating an unnecessary missing-data pixel.

NDVI change was then calculated as:

$$ \Delta NDVI = NDVI_{2023} - NDVI_{baseline} $$

The interpretation was:

- negative values indicated decreased vegetation greenness;
- positive values indicated increased vegetation greenness;
- values close to zero indicated relatively stable vegetation conditions.

A negative NDVI change was not interpreted automatically as permanent vegetation loss, because NDVI could also vary because of crop cycles, vegetation condition, land management, moisture differences, or other temporary effects.

# 6. NDVI validation in QGIS

The NDVI-change raster was exported from Google Earth Engine as a GeoTIFF and loaded into QGIS.

The raster was validated before it was included in the machine-learning dataset.

The following checks were performed:

- the raster aligned spatially with the Nyabarongo boundary;
- values outside the study area were represented as NoData/NaN;
- the raster contained continuous positive and negative values;
- the values were consistent with the theoretical NDVI range.

NDVI itself was expected to lie within:

$$ -1 \leq NDVI \leq 1 $$

Therefore, the theoretical change range was:

$$ -2 \leq \Delta NDVI \leq 2 $$

Most observed values were expected to lie much closer to zero.

The raster was also visually inspected to confirm that the spatial pattern was plausible.

# 7. Built-up percentage
## 7.1 Why BuiltPct was used instead of ISP

The original planned variable was Impervious Surface Percentage (ISP).

Impervious surfaces referred to surfaces such as roads, rooftops, concrete, and asphalt that prevented or reduced water infiltration.

A suitable 2023 impervious-surface product was not available at the required spatial and temporal resolution without introducing an additional classification model.

Creating a second machine-learning model solely to generate ISP was considered unnecessarily complex and could have introduced additional uncertainty into the flood-susceptibility model.

The variable was therefore changed from ISP to Built Percentage (BuiltPct).

BuiltPct was not treated as a direct measurement of physical imperviousness. Instead, it represented the proportion of each analysis cell classified as built-up land.

# 8. Dynamic World built-up data

Google Dynamic World was used to derive the 2023 built-up layer.

Dynamic World is a Sentinel-2-based, 10 m land-cover dataset available in Google Earth Engine.

Two relevant outputs were examined:

- label – the most likely land-cover class;
- built – the estimated probability that a pixel represented built-up land.

Within the Dynamic World label band:

```Built = class 6```

A July–December 2023 Dynamic World composite was created.

The modal land-cover label was used to determine the most frequently assigned class during the study period.

A binary built raster was then created:

```
1 = built
0 = not built
```

The built-probability layer was retained for visual validation and confidence assessment.

However, it was not used directly as the machine-learning predictor because an average classification probability was not equivalent to the physical percentage of a grid cell covered by built land.

The binary built classification provided a clearer and more reproducible quantity for spatial aggregation.

# 9. Built-up validation

Both the binary built raster and the built-probability raster were exported from GEE and inspected in QGIS.

The following checks were performed:

- binary values were restricted to 0 and 1;
- built-up locations appeared spatially plausible;
- the probability layer broadly supported the binary classification;
- pixels outside the study boundary were represented as NoData/NaN.

Only the binary layer was subsequently used to calculate BuiltPct.

# 10. Wetland loss to wetness decline
## 10.1 Original objective

The original objective was to derive a wetland-loss variable.

The intended interpretation was that loss or degradation of wetter land could reduce natural water storage and potentially increase flood susceptibility.

An initial definition classified wet-area loss where:

a pixel was wet during the baseline period;
the same pixel was no longer wet in 2023.

However, this approach was found to be too restrictive.

A pixel could have experienced a substantial drying signal without crossing a single wet/non-wet threshold.

For example, a pixel could have changed from an MNDWI of 0.30 to 0.10. This represented a clear decline in wetness, but both values remained above zero and the pixel would therefore not have been classified as wet-area loss.

The variable was consequently redefined as Wetness Decline, rather than ecological wetland loss.

This terminology was more appropriate because satellite spectral change alone could not confirm the permanent destruction of an ecological wetland.

# 11. MNDWI calculation

Wetness was measured using the Modified Normalized Difference Water Index:

$$ MNDWI = \frac{Green - SWIR}{Green + SWIR} $$

For Sentinel-2:

$$ MNDWI = \frac{B3 - B11}{B3 + B11} $$

where:

B3 represented green reflectance;
B11 represented short-wave infrared reflectance.

MNDWI was calculated for each individual cloud-masked Sentinel-2 image.

Median MNDWI composites were then created for:

- July–December 2018;
- July–December 2019;
- July–December 2023.

The 2018 and 2019 composites were combined to create the baseline.

A common valid-data mask was then applied so that change was calculated only where a valid baseline and a valid 2023 observation were available.

MNDWI change was calculated as:

$$ \Delta MNDWI = MNDWI_{2023} - MNDWI_{baseline} $$
# 12. Final wetness-decline definition

The final predictor identified significant drying within areas that had initially shown relatively wet or moist spectral conditions.

A pixel was classified as wetness decline where both of the following conditions were met:

$$ MNDWI_{baseline} > -0.10 $$

and

$$ \Delta MNDWI \leq -0.10 $$

The first threshold restricted the analysis to pixels that had shown at least some wet or moist characteristics during the baseline.

The second threshold required a substantial decrease in MNDWI rather than a very small fluctuation.

The resulting binary raster was:

```
 1 = significant wetness decline
 0 = no significant wetness decline
```

This variable was named wetness decline rather than wetland loss because the method identified a spectral drying signal rather than confirmed ecological wetland destruction.

# 13. Raster validation

The three final raster predictors were imported into QGIS:

- NDVI change;
- binary built-up area;
- binary wetness decline.

Each raster was checked for:

- correct spatial alignment;
- correct CRS;
- sensible value ranges;
- correct NoData/NaN handling;
- absence of unexpected values outside the study boundary.

These checks were performed before the variables were added to the machine-learning datasets so that erroneous spatial data were not introduced into the model.

# 14. Conversion from raster pixels to 200 m model variables

The machine-learning model operated on a 200 m × 200 m polygon grid, while the satellite-derived variables were raster datasets.

QGIS Zonal Statistics was therefore used to summarise raster pixels inside each grid cell.

## 14.1 NDVI change

For each 200 m grid cell, the mean NDVI change was calculated:

$$ NDVI\_Change_i = \frac{1}{n} \sum_{j=1}^{n} \Delta NDVI_j $$

The resulting value represented the average vegetation change within that grid cell.

Count, minimum, and maximum statistics were also calculated during validation.

Basic field statistics were used as a sanity check.

The final grid-level variable was:

```
NDVI_Change
```
## 14.2 Built percentage

The Dynamic World built raster contained only 0 and 1.

The mean of a binary variable was equivalent to the proportion of pixels classified as 1.

Therefore:

$$ BuiltPct = Mean(BinaryBuilt) \times 100 $$

For example:

```
Binary mean = 0.25
BuiltPct = 25
```

This indicated that approximately 25% of the 200 m grid cell had been classified as built-up.

## 14.3 Wetness-decline percentage

The wetness-decline raster was also binary.

The same approach was therefore used:

$$ WetDeclPct = Mean(BinaryWetnessDecline) \times 100 $$

For example:

```
Binary mean = 0.18
WetDeclPct = 18
```

This indicated that approximately 18% of the 200 m grid cell had met the significant wetness-decline condition.

The 20 m wetness raster produced approximately 100 raster pixels per 200 m grid cell.

The zonal-statistics validation showed that all 213,908 grid cells contained 100 valid wetness pixels, confirming that the large number of zero values represented genuine observed zeros rather than missing data.

# 15. Sequential QGIS grid construction

The new variables were added sequentially.

The intermediate grids were organised as follows:

```
Grid_01_NDVI
```

contained the original variables plus NDVI_Change.

```
Grid_02_Built
```

contained the original variables plus NDVI_Change and BuiltPct.

```
Grid_03_Wetness
```

contained the original variables plus all three new predictors:

```
NDVI_Change
BuiltPct
WetDeclPct
```

The final grid was saved as the updated Nyabarongo machine-learning prediction dataset.

# 16. Creation of matching training and prediction datasets

The machine-learning workflow required two datasets with matching predictor variables.

## Training dataset

The training dataset contained the original 302 flood and non-flood sample locations.

It contained:

- the original flood-conditioning variables;
- the target variable Flooded;
- NDVI_Change;
- BuiltPct;
- WetDeclPct.

Flooded = 0 represented a non-flooded sample location.

Flooded = 1 represented a flooded sample location.

## Prediction dataset

The prediction dataset contained the full 213,908-cell Nyabarongo grid.

It contained the same predictor variables as the training data but did not require the Flooded target field.

The trained model was later applied to this dataset to estimate flood susceptibility across the full study area.

# 17. Transfer of grid variables to the sample points

The new variables had first been calculated at the 200 m grid-cell scale.

The same grid-level values were then transferred to the original sample points using the QGIS Join Attributes by Location tool.

Each sample point inherited the values from the 200 m grid cell in which it was located.

This approach ensured that the new predictors had the same spatial meaning in both the training and prediction datasets.

The alternative approach of extracting a single 10 m or 20 m raster pixel at each sample point was not used because this would have introduced a mismatch between the spatial scale of the training data and the 200 m prediction grid.

# 18. Final data verification

Before machine-learning experiments were started, both datasets were checked.

The following were verified:

- the original predictor variables remained present;
- all three new predictor variables were present;
- the training dataset still contained the Flooded target;
- the prediction grid contained the same predictor fields but did not require a flood label;
- unexpected missing values were checked;
- the ranges of NDVI_Change, BuiltPct, and WetDeclPct were verified;
- the sample and prediction datasets contained matching predictor definitions.

At this stage, the GIS and Earth-observation feature-engineering workflow was considered complete.

# 19. Machine-learning Experiments

A separate Jupyter Notebook was created for each experiment.

The baseline experiment was first recreated to confirm that the original modelling workflow still reproduced the expected results.

The following experiments were then performed.

| Experiment       | Description                                                         | Purpose                                                                                             |
| ---------------- | ------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------- |
| **Baseline**     | Original flood-conditioning variables                               | Reproduced the original model and provided the reference result                                     |
| **Experiment 1** | Replaced `CN` with `BuiltPct`                                       | Tested whether built-up percentage could replace the existing curve-number representation           |
| **Experiment 2** | Retained `CN` and added `BuiltPct`                                  | Tested whether built-up information added predictive information beyond CN                          |
| **Experiment 3** | Retained `CN` and added `BuiltPct`, `NDVI_Change`, and `WetDeclPct` | Tested the effect of integrating the full set of new human-impact and land-surface-change variables |


The experiments were designed incrementally so that changes in model performance could be associated with specific additions to the predictor set rather than changing all variables simultaneously.

The results of these experiments were reported separately in the model-results and validation sections of the repository.

# 20. Summary of the workflow

The complete workflow was:

```
Original study-area grid and flood sample points
                    ↓
         CRS and data validation
                    ↓
        Dissolved study boundary
                    ↓
      Sentinel-2 processing in GEE
                    ↓
           NDVI change raster
                    ↓
      Dynamic World built raster
                    ↓
        Wetness decline raster
                    ↓
         Raster validation in QGIS
                    ↓
          QGIS zonal statistics
                    ↓
  NDVI_Change + BuiltPct + WetDeclPct
                    ↓
       Final 200 m prediction grid
                    ↓
     Spatial join to sample points
                    ↓
        Final training dataset
                    ↓
     Machine-learning experiments
                    ↓
       Flood susceptibility output
```

The overall purpose of this workflow was to extend the original flood-susceptibility model with interpretable Earth-observation indicators while preserving consistency between the training data and the full catchment prediction grid.

