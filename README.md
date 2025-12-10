# Grain Size Analysis of 316L Stainless Steel Microstructure

This repository contains an image-processing and deep-learning pipeline to estimate grain size distributions of additively manufactured 316L stainless steel from microstructure images. The workflow combines classical computer vision (thresholding, gradient filters, HED) and a U-Net CNN to segment grain boundaries and extract quantitative microstructural features.

## Dataset

The project uses a public grain-boundary dataset of 316L stainless steel produced by binder jetting additive manufacturing and imaged at 500× magnification. The original 1600×1200 images are split into tiles, and are paired with manually segmented masks (real grains) and synthetically generated microstructures (artificial grains, Voronoi-based). A calibration factor of 2.25 pixels per micron is used to convert measurements from pixels to physical units.

Main dataset folders:

- `Grains/`: reference micrographs of the real microstructure.
- `GRAIN DATA SET/AG`, `AGMask`: artificial grains and their ground-truth masks.
- `GRAIN DATA SET/RG`, `RGMask`: real grains and their ground-truth masks.
- `weights HED/`: Caffe model and deploy file used to run HED (Holistically-Nested Edge Detection).

Please respect the original dataset license and cite the authors if you use this project.

## Pipeline Overview

The main logic is implemented in:

- `notebooks/Finale notebook.ipynb`

The pipeline includes:

1. **Project setup and data access**
   - Definition of base paths and dataset structure.
   - Sanity checks on image shapes, counts, and file names.

2. **Pre-processing and calibration**
   - Loading representative micrographs.
   - Spatial calibration with 2.25 pixels per micron.
   - Preparation of tensors for training and evaluation.

3. **HED-based edge detection**
   - Loading the HED model from `weights HED/`.
   - Generating raw and filtered edge maps for each input image.
   - Saving outputs in `HED_raw/` and `HED_filtered/`.

4. **Supervised U-Net training**
   - Building a U-Net encoder–decoder architecture in Keras/TensorFlow.
   - Preparing training and validation sets from AG/RG images and masks.
   - Training with checkpoints, early stopping, and LR scheduling.
   - Monitoring loss and accuracy to assess convergence.

5. **Segmentation quality metrics**
   - Computing pixel-wise metrics: accuracy, precision, recall, F1-score, Dice, IoU.
   - Generating histograms and distributions of Dice/IoU per image.
   - Comparing performance between classical methods and U-Net.

6. **Label maps and feature extraction**
   - Converting binary masks to label maps using connected-component analysis.
   - Extracting per-grain features (area, perimeter, equivalent diameter, centroid,
     major/minor axes, axis ratio, circularity).
   - Storing results in structured CSV files and DataFrames.

7. **Physical units and grain-size distributions**
   - Converting pixel-based variables to microns and square microns using the
     calibration factor.
   - Building histograms, cumulative distributions, and summary statistics.
   - Producing planimetric-style summaries and CSV exports.

8. **Visualization and interactive interface**
   - Drawing overlays with colored grains and numeric IDs on the original images.
   - Providing an optional Gradio interface to:
     - Upload a micrograph,
     - Run segmentation,
     - Visualize labeled grains,
     - Download the corresponding table of grain features.

## Repository Structure

Suggested structure:

- `notebooks/`
  - `Finale notebook.ipynb` – main notebook containing the full pipeline.

- `data/`
  - `Grains/` – reference microstructure images.
  - `GRAIN DATA SET/` – artificial and real grain datasets with masks.
  - `weights HED/` – HED model weights and configuration.

- `models/`
  - `grains_unet_best.h5` – best U-Net weights (HDF5 format).
  - `grains_unet.keras` – U-Net saved in the native Keras format.

- `outputs/`
  - `HED_raw/`, `HED_filtered/`, `FilteredGradV2/`, `Segmented/` – processed images and masks.
  - CSV summaries:
    - `AR_HED_planimetric.csv`, `AREA_HED_planimetric.csv`, `CR_HED_planimetric.csv`,
      `PERIM_HED_planimetric.csv`, `MAX_HED_planimetric.csv`, `MIN_HED_planimetric.csv`
    - `Total_Grains_X_*.csv`, `Total_Grains_Y_*.csv`
    - `AVG_GrainX_*.csv`, `AVG_GrainY_*.csv`
    - `areas_grad_um2.csv`
    - `grain_features_with_units.csv`
