# AE Torch to Earth Engine

This folder contains a pure Earth Engine Python implementation of the trained
AlphaEarth building-height model.

## Files

- `model.py`: Earth Engine model code with embedded regional MLP weights.
- `ae_global_height_estimation.ipynb`: bbox-based height estimation demo.

The model uses the current best production setup:

- one model per region
- full 64 AlphaEarth channels
- plain height target
- no classification branch
- architecture: `64 -> 32(sigmoid) -> 1(relu)`

## Requirements

Run the notebooks from this directory so Python can import the local model file:

```python
import model as ae_height_model
```

The runtime environment needs:

- `earthengine-api`
- Earth Engine authentication
- access to `GOOGLE/SATELLITE_EMBEDDING/V1/ANNUAL`
- optional `geemap` for interactive maps

If your Earth Engine project is not `global-urban`, change:

```python
ee.Initialize(project="global-urban")
```

to your own project.

## Bbox Inference

For a bbox, use:

```python
bbox = [109.863281, 20.468189, 118.223877, 25.502785]
roi = ae_height_model.bbox_to_geometry(bbox)

region_candidates = ae_height_model.region_candidates_from_bbox(bbox)
roi_regions = [item["region"] for item in region_candidates]

height_roi = ae_height_model.predict_global_height(
    alpha_earth_image=alpha_earth,
    output_name="building_height_m",
    clip_geometry=roi,
    mask_source="lonlat",
    regions=roi_regions,
).clip(roi)
```

For China-only inference, `east_asia_pacific` is the expected regional model.

## Notes

- `predict_global_height()` can run all seven regional models, but bbox preview
  should pass `regions=...` to keep the Earth Engine graph small.
- Output height from `predict_height()` and `predict_global_height()` is in
  meters before any export encoding.
- For large production exports, tile-based export is recommended over
  interactive map preview.
