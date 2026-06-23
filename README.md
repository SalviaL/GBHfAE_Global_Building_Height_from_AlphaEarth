# AE Torch to Earth Engine

**Try the demo:**
https://global-urban.projects.earthengine.app/view/estimate-building-height-from-alphaearth
https://code.earthengine.google.com/4bc3869f00127fa94912835bce6905df

This folder contains the Earth Engine deployment code for the selected
AlphaEarth building-height model.

The deployed model is the current best setup from this project:

- one region, one model
- plain regression target, no classification branch
- full 64 AlphaEarth embedding bands
- MLP architecture: `64 -> 32(sigmoid) -> 1(relu)`
- output unit: meters

## Files

- `model.py`: pure Earth Engine Python implementation with embedded model
  weights for all seven regions.
- `ae_global_height_estimation.ipynb`: Python / geemap bbox demo using
  `model.py`.

### GEE code editor files
- `model_params.csv`: exported model parameters for the JavaScript workflow.
  Upload this CSV as an Earth Engine table asset.
- `gee_functions.js`: reusable GEE JavaScript functions. Save this as a Code
  Editor script module, then import it from `gee_main.js`.
- `gee_main.js`: GEE Web Code Editor / App demo. It lets users click two map
  anchors to define a bbox, then estimate building height inside that bbox.

## Accuracy

Metrics below are from `outputs/all_regions_mlp`, using held-out test samples.
This is the no-branch one-region-one-model MLP regression setup.

Overall weighted test performance across all regions:

- test samples: `114,500`
- weighted MAE: `0.586 m`
- weighted RMSE: `0.945 m`

| Region | Test N | MAE m | RMSE m | R2 |
|---|---:|---:|---:|---:|
| east_asia_pacific | 31,800 | 0.770 | 1.199 | 0.807 |
| europe_central_asia | 15,200 | 0.705 | 1.071 | 0.713 |
| latin_america_caribbean | 10,900 | 0.608 | 0.963 | 0.707 |
| middle_east_north_africa | 9,300 | 0.570 | 0.898 | 0.804 |
| north_america | 4,000 | 0.648 | 1.015 | 0.643 |
| south_asia | 25,900 | 0.343 | 0.584 | 0.862 |
| sub_saharan_africa | 17,400 | 0.487 | 0.700 | 0.744 |

## Python Usage

Run the notebook from this directory so Python can import the local module:

```python
import model as ae_height_model
```

The environment needs:

- `earthengine-api`
- Earth Engine authentication
- access to `GOOGLE/SATELLITE_EMBEDDING/V1/ANNUAL`
- optional `geemap` for map display

For bbox inference:

```python
bbox = [109.863281, 20.468189, 118.223877, 25.502785]
roi = ae_height_model.bbox_to_geometry(bbox)

region_candidates = ae_height_model.region_candidates_from_bbox(bbox)
roi_regions = [item["region"] for item in region_candidates]

height = ae_height_model.predict_global_height(
    alpha_earth_image=alpha_earth,
    output_name="building_height_m",
    clip_geometry=roi,
    mask_source="lonlat",
    regions=roi_regions,
).clip(roi)
```

For China, the expected model region is `east_asia_pacific`.

## GEE Web App Usage

Upload `model_params.csv` as an Earth Engine table asset, then update this line
in `gee_main.js`:

```javascript
var PARAMS_ASSET = 'projects/global-urban/assets/parameters/model_params';
```

Save `gee_functions.js` as a GEE Code Editor script module. The current
`gee_main.js` imports it with:

```javascript
var ae = require('users/salvialu96/AE-Height:gee_functions');
```

Make sure both the script module and the parameter table asset are readable by
the published App.

The App interaction is:

1. Click `Draw bbox`.
2. Click the first map anchor.
3. Click the opposite anchor.
4. Click `Estimate height in bbox`.

The map displays only positive predicted heights:

```javascript
var heightVis = height.updateMask(height.gt(0));
```

The ROI is shown as a red outline.

## Export Encoding

`gee_functions.js` includes:

```javascript
var encoded = encodeUint8Dm(height);
```

This stores height as unsigned 8-bit decimeters:

- exported DN = `round(height_m * 10)`
- decoded height in meters = `DN / 10`
- values are clipped to `[0, 25.5]` meters

Use this compact encoding when Drive export size matters and decimeter precision
is sufficient.

## Notes

- AlphaEarth inputs are expected to already be in the same normalized embedding
  space used for training; no extra input normalization is applied here.
- `predict_global_height()` can run all seven regional models, but bbox demos
  should pass only the needed regions to keep the Earth Engine graph smaller.
- For large production jobs, prefer tiled exports over interactive map preview.
