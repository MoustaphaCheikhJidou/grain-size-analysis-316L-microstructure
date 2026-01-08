# Grain Size Estimation for 316L Stainless Steel Microstructure (Segmentation & Metrology)

This repository provides an image-processing and deep-learning pipeline to estimate grain-size information from micrographs by combining classical approaches (thresholding, gradient/Canny, HED) with a supervised U-Net segmentation model, followed by metrology (morphometric features + conversion to physical units).

## Data (public sources)

The datasets used in this project are publicly available on Kaggle:

- Real micrographs (316L ExONE, 500×): https://www.kaggle.com/datasets/peterwarren/exone-stainless-steel-316l-grains-500x
- Artificial grains (Voronoi): https://www.kaggle.com/datasets/peterwarren/voronoi-artificial-grains-gen

The real dataset corresponds to 316L stainless steel produced by Binder Jetting additive manufacturing (ExONE) and imaged with an optical microscope at 500× magnification. Initially, 21 micrographs of size 1600×1200 pixels are split into 400×300 tiles. The artificial dataset is based on Voronoi-generated microstructures with exact masks.

## Folder layout (BASE_DIR)

Under the project root `BASE_DIR` (mounted from Google Drive in the notebook), the main folders are:

- `Grains/`: reference micrographs (400×300 tiles).
- `HED_raw/` and `HED_filtered/`: raw and filtered HED edge maps.
- `FilteredGradV2/`: gradient-based outputs (version 2).
- `Segmented/`: segmentation results (thresholding, gradient, HED, U-Net).
- `weights HED/`: `deploy.prototxt` and `hed_pretrained_bsds.caffemodel`.
- `GRAIN DATA SET/`: supervised dataset for U-Net, organized as:
  - `AG/` and `AGMask/`: artificial grains (Voronoi) and masks.
  - `RG/` and `RGMask/`: real grains and ground-truth masks.
  - `THRESH_PRE/`, `HED_PRE/`, `GRAD_PRE/`: preprocessed variants (priors/inputs).

## Spatial calibration (pixel → microns)

The conversion to physical units is based on a scale bar: **225 pixels for 100 microns**, hence:

- `scale = 225/100 = 2.25` (pixels per micron)
- `P2M = 1/scale` (microns per pixel)

This calibration is used to convert areas (pixels → µm²) and lengths (pixels → microns) in the metrology outputs.

## Pipeline overview

The workflow is structured as follows:

1. **Initialization & project structure**
   - Environment setup, definition of `DRIVE_ROOT` and `BASE_DIR`, and organization of input/output/model folders.

2. **Data preparation & calibration**
   - Image preparation for downstream steps and definition of `P2M` for conversion to physical units.

3. **Classical segmentation (threshold & gradient)**
   - Manual thresholding + median blur via `manual_threshold_and_blur`.
   - Canny-based edge detection + binarization via `segment_gradient`.
   - Outputs: binary masks/edges usable for labeling and measurement.

4. **Planimetric estimation (Line Intercept)**
   - Regular grid with `N_LINES_GRID = 20` and intercept counting via discrete differences (`np.diff`).
   - Outputs: `Total_Grains_X`, `Total_Grains_Y` and averages `AVG_GrainX`, `AVG_GrainY`.

5. **HED segmentation and priors**
   - HED edge detection from `deploy.prototxt` and `hed_pretrained_bsds.caffemodel` (with `CropLayer`).
   - Production of binary masks and an `hed_prior` usable by U-Net.

6. **Supervised U-Net (image + priors)**
   - Training on supervised data (256×256, 1 channel: `IMG_HEIGHT=256`, `IMG_WIDTH=256`, `IMG_CH_IMG=1`).
   - Model configuration includes `U_Net_GrainSeg_GradHED` and priors (e.g., `grad_prior`, `hed_prior`).
   - Loss/metrics: `bce_dice_loss`, `dice_coef`, `MeanIoUThreshold` (as `iou_score`), plus binary metrics (`bin_acc`, `precision`, `recall`).

7. **Labeling, morphometry & reporting**
   - Labeling via `label_components` (based on `cv2.connectedComponentsWithStats`) and area filtering via `filter_components_by_area`.
   - Morphometric extraction via `compute_grain_features_from_labels` (using `regionprops_table`), with fallback via `fallback_features_from_cc`.
   - Contour-based measurements via `measure_grains_planimetric` (area, perimeter, circularity, aspect ratio, Feret-type distances) and unit conversion via `add_physical_units`.
   - Reporting/inspection with visualization (e.g., `draw_grain_ids_on_image`) and a Gradio interface `segment_and_measure` offering `"U-Net"`, `"HED"`, `"Gradient"`, `"Threshold"` to output an overlay and a measurement table.

## Key variables (summary)

Central variables used for identification, morphometry, calibration, and indicators:

- Identifiers: `image_id`, `grain_id`
- Pixel-based measurements: `area_px`, `perimeter_px`, `equiv_diameter_px`, `centroid_row`, `centroid_col`, `major_axis_px`, `minor_axis_px`, `axis_ratio`, `circularity`
- Calibration & physical units: `P2M`, `area_um2`, `equiv_diameter_um`, `major_axis_um`, `minor_axis_um`
- Quality/quantification indicators: `dice_score`, `iou_score`, `Total_Grains_X`, `Total_Grains_Y`, `AVG_GrainX`, `AVG_GrainY`

## Running the project

The main logic is implemented in the project notebook (executed in Google Colab with Google Drive mounted). A typical run is:

1. Mount Google Drive.
2. Set `BASE_DIR` and ensure the expected folders exist.
3. Run segmentation (classical, HED, U-Net), then labeling and measurement extraction.
4. Generate reporting outputs (overlays/visualizations and measurement tables).

