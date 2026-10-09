# Measure Your Cricket

Automated morphometric measurement of house crickets (*Acheta domesticus*) from photographs, using [DeepLabCut](https://github.com/DeepLabCut/DeepLabCut) pose estimation.

This pipeline trains a keypoint model on labelled cricket images, runs it on new photographs, and returns eye width, thorax width and thorax length for each animal, with low-confidence predictions flagged for manual review.

It was developed for Chapter 1 of the PhD dissertation *Crickets Under Pressure: Integrated Computer Vision and Transcriptomic Analysis of Stress Responses for Commercial Cricket Production on Earth and in Space* (Emily McColville, Carleton University, 2026; MacMillan and Bertram Labs).

## Contents

| File | Purpose |
|------|---------|
| `supplement1A_data_prep.ipynb` | Sample training images from raw photo folders, standardise their size, and copy them into the DeepLabCut project |
| `supplement1B_model_training.ipynb` | Build the training dataset, train the network, resume from a snapshot, and evaluate |
| `supplement1C_model_testing.ipynb` | Run the trained model on new images, calculate distances, and export annotated images plus a measurements spreadsheet |
| `fix_the_scale.ipynb` | Ignore - script to fix a mistake I made while developing (scale bar correction) |

The notebooks are written for Google Colab with the project stored on Google Drive. They can run locally with minor path changes.

## Requirements

- Python 3 (Colab default)
- `deeplabcut==2.3.10`
- `tensorflow==2.12`
- `tensorpack`
- `pandas`, `numpy`, `matplotlib`, `Pillow`, `tqdm`, `XlsxWriter`

Each notebook installs its own dependencies in the first cell. A GPU runtime is strongly recommended for 1B and 1C.

## Keypoints

The model predicts six landmarks:

| Keypoint | Location |
|----------|----------|
| `eye_right`, `eye_left` | Outer edge of each compound eye |
| `thorax_right`, `horax_left` | Lateral edges of the pronotum |
| `thorax_top`, `thorax_bottom` | Anterior and posterior edges of the pronotum |

Note: `horax_left` is a typo that was baked into the trained model's `config.yaml`. The code preserves it deliberately so the keypoint names match. If you retrain from scratch, you can correct it in the config before labelling.

## Workflow

### 1. Prepare training data (`supplement1A`)

1. Set `input_folders` to the folders holding your raw photographs and `output_folder` to where sampled images should go.
2. Run the sampling cell. It randomly selects `num_images` (default 50) from each source folder and prefixes each filename with its source folder name to avoid collisions.
3. Check the image size distribution. DeepLabCut expects consistent dimensions.
4. Resize all images to the smallest width and height found, save as lowercase `.png`, and copy them into `labeled-data/<video_name>/` inside the DeepLabCut project.
5. A second block selects an additional 30 unseen images for validation, avoiding duplicates from the first pass.

Labelling itself is done outside these notebooks with the DeepLabCut GUI (`deeplabcut.label_frames(config_path)`), which does not run in Colab.

### 2. Train the model (`supplement1B`)

1. Point `config_path` at your project's `config.yaml`.
2. `create_training_dataset` builds the train/test split (95% train by default).
3. `train_network` runs for up to 200,000 iterations, saving a snapshot every 5,000.
4. To resume after a Colab timeout, use the "Continue Training" section, which restarts from the most recent snapshot.
5. `evaluate_network` reports train and test pixel error and plots predictions on the held-out images.

### 3. Measure new crickets (`supplement1C`)

1. Drop your photographs into `measure-your-cricket/put-photos-here/` inside the project folder. They should be `.png` and match the training image dimensions.
2. Run the image size check to confirm.
3. `analyze_time_lapse_frames` runs inference and writes an `.h5` and `.csv` of keypoint coordinates and likelihoods.
4. The measurement cell then, for every image:
   - computes Euclidean distance (in pixels) for `eye_width`, `thorax_width` and `thorax_length`
   - draws the keypoints and connecting lines on the image and saves it to `results/annotated-images/`
   - copies the annotated image to `results/flagged-for-manual-review/` if any keypoint in a pair has likelihood below 0.95
   - writes `results/annotated-images/measurements.xlsx`, with per-keypoint confidence columns and flagged values shown in red text

### 4. Convert to real units (`fix_the_scale`)

Resizing in step 1 changes the number of pixels per micrometre. This notebook takes a spreadsheet with the original scale bar length (µm and px) and the original and resized image widths, and computes:

```
original_um_per_px = orig_scale_bar_um / orig_scale_bar_px
resize_ratio       = resized_width / orig_width
new_um_per_px      = original_um_per_px / resize_ratio
```

Multiply the pixel distances from step 3 by `new_um_per_px` to get measurements in µm.

## Output structure

```
<dlc-project>/
└── measure-your-cricket/
    ├── put-photos-here/           # your input images (+ .h5/.csv from DLC)
    └── results/
        ├── annotated-images/      # every image with keypoints drawn
        │   └── measurements.xlsx  # distances (px) and confidences
        └── flagged-for-manual-review/   # copies of low-confidence images
```

## Known quirks

- All paths are hard-coded to a specific Google Drive layout. Update them before running.
- Some cells reference two different project folders (`cricket_measurements_v1-emily-2025-01-13` and `cricket-measurements-emily-2025-03-03`). These were two training iterations; use whichever `config.yaml` you are working with.
- The trained model weights and labelled data are not included in this repository.

## Citation

If you use this pipeline, please cite:

> McColville, E. (2026). *Crickets Under Pressure: Integrated Computer Vision and Transcriptomic Analysis of Stress Responses for Commercial Cricket Production on Earth and in Space* [Doctoral dissertation, Carleton University].

and DeepLabCut:

> Mathis, A., Mamidanna, P., Cury, K.M. et al. (2018). DeepLabCut: markerless pose estimation of user-defined body parts with deep learning. *Nature Neuroscience* 21, 1281–1289.

## Licence

Add a licence file (MIT is a common choice for research code).
