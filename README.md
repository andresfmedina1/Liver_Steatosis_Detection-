# Liver_Steatosis_Detection-


This project implements a **semantic segmentation pipeline** for liver steatosis vacuoles on histology patches using a **U-Net** model, with an **ablation study** over preprocessing, training settings, and post-processing.

The workflow is organized as **offline preprocessing → training → threshold selection → test inference + post-processing → evaluation**.

---

## Project contents

```
Group_AS11_Challenge_Liver/
├─ Group_AS11_Preprocessing.ipynb     # Offline normalization datasets (ABL-1..ABL-8)
├─ Group_AS11_Training.ipynb          # U-Net training + ablations + threshold sweep
├─ Group_AS11_Mockup.ipynb            # Final test inference + evaluation + debug figures
├─ Report_AS11.pdf                    # Final report (method + ablations + results)
├─ FINAL_MODEL/
│  ├─ best_model.pt                   # Final trained weights (example export)
│  ├─ best_threshold.txt              # Best validation threshold (example export)
│  ├─ training_params.json            # Final model/training configuration
│  ├─ mask/                           # Saved predicted masks (0/255)
│  ├─ metrics_summary.*               # Summary metrics (JSON/TXT)
│  ├─ per_image_metrics.csv           # Per-image metrics
│  └─ debug_comparisons/              # Panels/overlays for qualitative inspection
├─ test/                              # Example “professor test” structure (image + GT)
└─ test_preprocessed/                 # Example preprocessed test image folder
```

> Note: The archive includes a **minimal test example** (one patch) to demonstrate the complete pipeline end-to-end.
> The full ablation results are reported in **Report_AS11.pdf**.

---

## Requirements

Tested in **Google Colab** (recommended). Main dependencies:
- Python 3.9+
- `torch`, `torchvision`
- `numpy`, `opencv-python`, `Pillow`
- `scipy`, `pandas`, `tqdm`
- (Optional) `matplotlib` for plots/figures

---

## Dataset structure expected

The code assumes datasets follow this layout:

```
DATASET_xxx/
├─ train/
│  ├─ image/        (*.png / *.jpg ...)
│  └─ manual_py/    (binary masks, preferred)
├─ val/
│  ├─ image/
│  └─ manual_py/
└─ test/
   ├─ image/
   ├─ manual_py/    (optional; if missing, fallback to manual/)
   └─ manual/
```

Masks are expected to be binary-like images (commonly stored as **0/255**).

---

## Step-by-step workflow

### 1) Offline preprocessing (normalization strategies)
Notebook: **`Group_AS11_Preprocessing.ipynb`**

This notebook creates *new dataset folders* with different normalization strategies for the ablation study.
Each strategy writes a new `DATASET_liver_stu_*` folder with the same `train/val/test` structure and copies masks unchanged.

Implemented variants (names match the ablation table/report):

- **ABL-1**: LAB-based luminance standardization (L only)
- **ABL-2**: LAB-based luminance standardization (alternative parameters)
- **ABL-3**: CLAHE on LAB-L (local contrast)
- **ABL-4**: Reinhard normalization in LAB using a target reference patch
- **ABL-5**: Soft Macenko (λ=0.35) using a target stain reference
- **ABL-6**: Soft Macenko (λ=0.20) using a target stain reference *(selected as the most conservative)*
- **ABL-7**: Multi-target soft Macenko (3 targets, random selection per image + low-tissue fallback)
- **ABL-8**: Soft Vahadane-like normalization (sparse NMF in OD + soft blending)

**Important:** Target images are always selected from **train/image/** (tissue-rich and representative).

---

### 2) Training U-Net (with ablations)
Notebook: **`Group_AS11_Training.ipynb`**

Key components:
- **Model:** U-Net (encoder–decoder) with configurable `init_filters` and `depth`
- **Input:** RGB images resized to a fixed `INPUT_SIZE` (default 416×416)
- **Loss options:** Dice, BCE+Dice, Focal Tversky, BCE+Focal Tversky
- **Optimizer:** Adam / AdamW (ablation)
- **Early stopping:** optional, based on validation
- **(Optional) sampler:** for weighted sampling (ablation)
- **Post-processing during validation:** optional, to compare raw vs cleaned masks

Outputs:
- `best_model.pt` (checkpoint with best validation performance)
- training logs + history plots
- threshold sweep artifacts (see next step)

---

### 3) Threshold selection (validation sweep)
Inside **`Group_AS11_Training.ipynb`** (dedicated block)

After training, the notebook loads `best_model.pt` and sweeps thresholds on the validation set
(e.g., 0.20 → 0.80 step 0.05), selecting the threshold that maximizes mean Dice.

Outputs:
- `best_threshold.txt`
- `threshold_sweep.json` / `threshold_sweep.csv`

---

### 4) Final inference + evaluation (professor-style test)
Notebook: **`Group_AS11_Mockup.ipynb`**

This notebook:
1. Loads the trained model (`best_model.pt`) + the chosen threshold (`best_threshold.txt`)
2. Runs inference on `test/image/`
3. Applies **post-processing** (optional; enabled in the final configuration)
4. Saves final predicted masks as 0/255 images
5. Evaluates predictions using the provided test masks (prefers `manual_py/`, else `manual/`)
6. Produces qualitative debug panels/overlays

---

## Post-processing options (used after thresholding)

The project tests several post-processing strategies:
- **min_area:** removes small connected components (noise)
- **min_area_fill:** removes small objects + fills holes inside objects *(final choice in FINAL_MODEL)*
- **watershed:** attempts to split merged vacuoles using distance transform + watershed
- **region_growing:** seeds from high-confidence pixels and grows with a lower threshold

---

## Challenge metrics implemented

The evaluation script reports the required metrics:

1. **Dice Similarity Coefficient (DSC)**  
2. **Count error**: absolute difference in number of connected components  
3. **Steatosis percentage error**: absolute difference in foreground area percentage  

Additional metrics (for analysis) include IoU, pixel precision/recall, and object-level F1.

---

## Reproducing the final run (quick start)

1) Open `Group_AS11_Preprocessing.ipynb` and generate the selected dataset variant (e.g., **ABL-6**).  
2) Open `Group_AS11_Training.ipynb`:
   - set `DATASET_NAME` to the desired preprocessed dataset folder
   - run training blocks to produce `best_model.pt`
   - run the threshold sweep block to produce `best_threshold.txt`
3) Open `Group_AS11_Mockup.ipynb`:
   - point `FINAL_MODEL_DIR` to the folder containing `best_model.pt`, `best_threshold.txt`, `training_params.json`
   - run inference + evaluation

---

## Final artifacts included

The `FINAL_MODEL/` folder in this archive contains an example export:
- `best_model.pt`, `best_threshold.txt`, `training_params.json`
- saved predicted mask(s) and evaluation summaries
- debug panels/overlays for qualitative checks

For the full experimental results and ablation discussion, see **`Report_AS11.pdf`**.

---

## Authors / Group

Group AS11 (Medical Image Processing — Challenge Liver Steatosis).
