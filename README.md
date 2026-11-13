# QRShield v2

**A Benchmark for Visual Tamper Detection in QR Codes Beyond the Training Distribution**

Version 2.0 | November 2026 | Dataset and report: CC BY 4.0 | Code: MIT
Author: <Memona hafeez> (ORCID <0009-0007-4258-2386>) | Contact: <memonahafeez12@gmail.com>
Dataset DOI: <ZENODO DOI> | Code: <GITHUB URL>

---

## Abstract

QR codes can be tampered with after printing, for example by overlaying a sticker, altering modules or substituting a logo, and a visual detector should flag such changes. Our first detector (QRShield v1) scored 100% on data resembling its training set, but only 84.9% on camera-degraded images and 51.7% accuracy on tamper types it had never seen. This release addresses that generalisation failure. We introduce QRShield v2, a synthetic benchmark of 66,083 images with five tamper families, styled and logo-bearing benign codes as hard negatives, and a camera-degradation layer, together with a leakage-free evaluation protocol (base-code group split, leave-one-family-out testing, validation-only thresholding, three seeds). A fine-tuned EfficientNetB0 reaches 0.969 AUROC in distribution, 0.927 on a camera-degraded benchmark and 0.854 on held-out tamper families, against 0.616, 0.589 and 0.554 for a v1-style baseline. A one-factor diagnostic attributes most of the v1 failure to unseen benign variation (styled codes, camera effects) rather than to unseen tamper types. All images are synthetic and no real-camera evaluation is included, so the reported gains describe the benchmark, not deployed performance.

## Contents

1. [Motivation and contributions](#1-motivation-and-contributions)
2. [Dataset](#2-dataset)
3. [Evaluation protocol](#3-evaluation-protocol)
4. [Methods](#4-methods)
5. [Results](#5-results)
6. [Limitations](#6-limitations)
7. [Future work](#7-future-work)
8. [Reproducibility](#8-reproducibility)
9. [Repository layout](#9-repository-layout)
10. [Ethics and intended use](#10-ethics-and-intended-use)
11. [Citation, license and acknowledgements](#11-citation-license-and-acknowledgements)

---

## 1. Motivation and contributions

QRShield is a layered defence against malicious QR codes. Layer 1 is a visual tamper detector, and the other layers verify signatures and TLS. This repository concerns Layer 1 only.

The v1 detector reported near-perfect accuracy on data drawn from its own training distribution, but that number did not predict behaviour under camera degradation or on new tamper types. The research question of this release is therefore: **what training data and evaluation design make a visual QR tamper detector generalise, and how can generalisation be measured without leakage?**

Contributions:

1. **A benchmark** (QRShield v2): 66,083 synthetic images, five tamper families, two classes of hard-negative benign codes, and a camera-degradation layer applied to both classes.
2. **An evaluation protocol** that removes base-code leakage and measures generalisation to unseen tamper families.
3. **A factor-isolating diagnostic** that separates the effects of camera degradation, benign styling and unseen tamper families on the v1 failure.
4. **Baselines and robustness analyses**: fine-tuned and frozen-feature models, adversarial attacks, and a companion study of frequency-domain methods, fusion, noise, distribution shift and transfer, including negative results.

## 2. Dataset

### 2.1 Definition

A tamper is any modification applied to a legitimate code after generation that alters, hides or replaces it. A logo added at generation time within the error-correction capacity is benign.

### 2.2 Composition

The dataset is built from 12,000 base QR codes with about five images each. It comprises a 60,000-image main set and a separate 6,083-image camera benchmark, 66,083 images in total, stored as 24 shards of 500 base codes.

| Class | Type | Train | Val | Test | Total |
|---|---|---|---|---|---|
| benign_plain | benign | 8,398 | 1,165 | 2,437 | 12,000 |
| benign_styled (rounded or coloured modules) | benign, hard negative | 6,280 | 878 | 1,842 | 9,000 |
| benign_styled_logo (legitimate logo) | benign, hard negative | 6,301 | 862 | 1,837 | 9,000 |
| tamper_module_corruption | tamper | 4,176 | 586 | 1,238 | 6,000 |
| tamper_patch_overlay | tamper | 4,185 | 564 | 1,251 | 6,000 |
| tamper_logo_manipulation | tamper | 4,195 | 570 | 1,235 | 6,000 |
| tamper_sticker_overlay (attacker QR on top) | tamper | 4,197 | 599 | 1,204 | 6,000 |
| tamper_occlusion_warp | tamper | 4,226 | 586 | 1,188 | 6,000 |

The main set has 41,958 training, 5,810 validation and 12,232 test images, balanced at 50% benign and 50% tampered.

### 2.3 Camera layer

Perspective, blur, glare, moire, JPEG compression, shadow and sensor noise are applied to benign and tampered images alike, so that degradation cannot serve as a shortcut cue for the label. About 40% of main-set images (40.4% of benign, 40.1% of tampered) receive a mild version. The 6,083-image camera benchmark is built only from test base codes, receives strong degradation, and is used for evaluation only.

### 2.4 Manifest

Every image has a manifest row with its base ID, class, family, label, split, applied camera operations, and OpenCV decode flags (whether the code still decodes, and whether to the original or a different payload). Column names: `<adjust to the real manifest.csv header>`.

### 2.5 Safety

All payloads are harmless placeholder URLs on reserved example domains. No real brand logos are used.

## 3. Evaluation protocol

- **Group split.** Train, validation and test are split by base QR ID (70/10/20), so that images derived from one base code never occur in two splits. This removes a leakage route that can inflate in-distribution scores.
- **Leave-one-family-out (LOFO).** One tamper family is removed from training and validation, and the model is tested on that family together with benign test images. This is repeated for all five families.
- **Thresholding.** The decision threshold maximises balanced accuracy on the validation set and is never tuned on test data.
- **Metrics.** AUROC (primary, threshold-free), accuracy, balanced accuracy and F1, per family, as mean and standard deviation over seeds 0, 1 and 2. In per-family evaluation, benign images are subsampled to the family size with a fixed seed.

We ask that results on this benchmark be reported under the same protocol.

## 4. Methods

| Method | Description |
|---|---|
| v1 (reproduction) | EfficientNetB0 on the v1 setting: plain benign codes, two tamper families (module corruption, patch overlay), no camera layer, 9,968 training images |
| aug | The v1 data with augmentation (blur, brightness and contrast, noise, JPEG, small affine changes) |
| **v2** | ImageNet-pretrained EfficientNetB0 on the full v2 training set: classifier head for 3 epochs, then the top 30 layers for 8 epochs (about 12 minutes per model on a T4 GPU) |
| freq | 81 global frequency features (radial and angular FFT profile, high-frequency energy ratio) with gradient boosting |
| dino | Frozen DINOv2-small features (CLS token and mean patch token) with a small MLP; the pooling choice was selected by validation LOFO AUROC |

The v1 baseline is a reproduction of the v1 data and recipe, not the original weights (see Section 6).

## 5. Results

### 5.1 Generalisation to camera degradation and unseen tamper families

**Table 1. AUROC (mean over three seeds).**

| Test | v1 (repro.) | aug | **v2** | freq | dino |
|---|---|---|---|---|---|
| In-distribution test | 0.616 | 0.623 | **0.969** | 0.818 | 0.960 |
| Camera-degraded benchmark | 0.589 | 0.666 | **0.927** | 0.674 | 0.886 |
| Unseen tamper families (LOFO mean) | 0.554* | 0.567* | **0.854** | 0.766 | 0.840 |

\* v1 and aug averages cover three unseen families, the other methods cover five. On the same three families the v2 LOFO average is 0.870. Seed standard deviations are small (0.000 to 0.031).

Training on the full v2 data improves in-distribution AUROC from 0.616 to 0.969, camera-benchmark AUROC from 0.589 to 0.927 and held-out-family AUROC from 0.554 to 0.854. The in-distribution value for v1 is far below its published 100% because the v2 test set contains styled and logo-bearing benign codes that v1 never saw and because the split is grouped by base code; Section 5.2 shows where that gap comes from. Accuracy at the validation threshold follows the same ordering (in-distribution: v2 0.940, dino 0.917, freq 0.747, v1 0.601; camera benchmark: v2 0.827, dino 0.808, freq 0.614, v1 0.502).

**Table 2. AUROC on each tamper family when it is held out of training.**

| Held-out family | v1 | aug | **v2** | freq | dino |
|---|---|---|---|---|---|
| sticker_overlay | 0.412 | 0.383 | **0.941** | 0.850 | 0.902 |
| module_corruption | 0.571* | 0.576* | 0.661 | 0.575 | 0.634 |
| patch_overlay | 0.845* | 0.837* | **0.999** | 0.847 | 0.997 |
| logo_manipulation | 0.703 | 0.780 | 0.759 | 0.768 | 0.764 |
| occlusion_warp | 0.546 | 0.540 | **0.911** | 0.787 | 0.902 |
| Average | 0.554 | 0.567 | **0.854** | 0.766 | 0.840 |

\* family was in the training set of v1 and aug.

Generalisation is uneven across families. Families with a large visual footprint (patch overlay, sticker overlay, occlusion and warp) transfer well under v2 (0.911 to 0.999). Module corruption (0.661) and logo manipulation (0.759) do not, and on logo manipulation the augmentation-only baseline (0.780) is slightly ahead of v2. In these two families a legitimate change and a malicious change are visually close. Ranking quality also transfers better than a fixed operating point: on module corruption the v2 F1 at the validation threshold is only 0.252. Averaged over unseen families, v1 attains a higher F1 (0.706) than v2 (0.666), but this reflects over-flagging (Section 5.2): balanced accuracy is 0.604 for v1 against 0.773 for v2.

A frozen DINOv2 feature model with a small MLP comes within 0.009 AUROC of fine-tuned EfficientNetB0 in distribution and within 0.014 on unseen families, at a fraction of the training cost. Global FFT features alone are clearly weaker (0.766 on unseen families), and are near chance on module corruption (0.575).

### 5.2 Why v1 failed: isolating one factor at a time

The headline v1 versus v2 gap mixes three differences: the camera layer, styled and logo-bearing benign codes, and unseen tamper families. We evaluated saved models on test subsets that vary one factor at a time.

**Table 3. AUROC by isolated factor.**

| Factor | v1 | aug | v2 |
|---|---|---|---|
| A. v1's own data type (plain benign, two families, no camera) | 0.974 | 0.959 | 0.959 |
| B. Same classes with the camera layer | 0.869 | 0.945 | 0.941 |
| C. Styled and logo benign codes as negatives, no camera | 0.573 | 0.521 | 0.932 |
| D. Unseen tamper families against plain benign, no camera | 0.986 | 0.860 | 0.996 |

**Table 4. False-positive rate on benign test codes (lower is better).**

| Benign type | v1 | aug | v2 |
|---|---|---|---|
| plain, clean | 0.015 | 0.010 | 0.003 |
| plain, camera | 0.934 | 0.247 | 0.014 |
| styled, clean | 0.980 | 0.822 | 0.003 |
| styled, camera | 1.000 | 0.917 | 0.015 |
| styled with logo, clean | 1.000 | 0.998 | 0.028 |
| styled with logo, camera | 1.000 | 0.998 | 0.067 |

v1 performs well on its own kind of data (A) and, notably, also on unseen tamper families when the benign codes are plain and clean (D, 0.986). Its collapse is driven by factor C: styled and logo-bearing benign codes drop its AUROC to 0.573, with 98 to 100% of such codes labelled as tampered. Camera effects add a second failure (B, and a 0.934 false-positive rate on plain camera images). Augmentation repairs the camera effect on plain codes (0.934 to 0.247) but not the styling effect (0.822 to 0.998). Consequently, the high tamper recall of v1 should not be read as good detection, because it flags almost everything. The result supports the view that coverage of benign variation in training, rather than tamper-type diversity alone, drives the improvement. It does not isolate the contribution of tamper-family diversity, which requires the controlled baseline in Section 7.

### 5.3 Adversarial robustness

White-box digital attacks (FGSM and 10-step PGD, step size eps/4) were run on the final v2 models, using 1,500 test images per seed, with L-infinity budgets in units of 1/255.

**Table 5. Accuracy under attack (mean over three seeds).**

| eps | FGSM accuracy | PGD accuracy | Tampered images missed under PGD |
|---|---|---|---|
| 0 (clean) | 0.937 | 0.937 | 0.106 |
| 1 | 0.350 | 0.193 | 0.628 |
| 2 | 0.269 | 0.045 | 0.910 |
| 4 | 0.237 | 0.006 | 0.987 |
| 8 | 0.256 | 0.007 | 0.986 |

The model has no adversarial defence, and a perturbation of 1/255 already halves its accuracy under FGSM, while PGD at 4/255 misses 98.7% of tampered images. AUROC falls below chance under attack, which indicates that the attacks invert the learned ranking rather than merely add noise. These are digital attacks, so physical feasibility is not established. They do show that the visual layer alone cannot be relied upon, which motivates the layered design in which signature verification covers cases an image detector cannot.

### 5.4 Companion study: frequency priors, fusion, noise, shift and transfer

Six questions were posed on the full dataset, adapted from methods in the HFP-SAM line of work (Prof. Pingping Zhang's group). HFP-SAM (arXiv 2603.12708) was read in full. SFMFusion (arXiv 2511.06593) and ISFM (arXiv 2602.04405) were read only at abstract and introduction level, so RQ5 is a simplified analogue of the interaction idea and not a reproduction. Deviations: windows do not overlap; the high-frequency map averages the magnitudes of the three Haar sub-bands; classifiers act on frozen features instead of end-to-end segmentation networks; MMD is unbiased RBF-MMD with the median heuristic; Wasserstein-1 is a sliced estimate with 64 projections.

**RQ1 and RQ2. Does a local wavelet prior beat the global spectrum, and does window selection matter?**

**Table 6. AUROC for frequency feature sets (same classifier and protocol).**

| Features | Seen families (avg) | Unseen families (LOFO avg) | Unseen module_corruption |
|---|---|---|---|
| fft_global (81-d) | 0.819 | 0.766 | 0.575 |
| wav_topk (top-k high-frequency windows) | 0.748 | 0.701 | 0.554 |
| wav_multi (multi-scale) | 0.786 | 0.742 | 0.579 |
| wav_random (random windows) | 0.770 | 0.715 | 0.568 |
| wav_global (all windows) | 0.749 | 0.704 | 0.560 |
| canny_topk | 0.683 | 0.644 | 0.540 |
| dino_ref (reference) | 0.960 | 0.840 | 0.634 |

The hypothesis that a local frequency prior helps local tampering was not supported: no wavelet variant beats global FFT features on unseen families, including on module corruption. Selecting windows by high-frequency energy gave no benefit over random windows (0.770 random against 0.748 top-k on seen families), and the number of selected windows is irrelevant (k = 5, 10, 20 give 0.749, 0.748, 0.747). Larger windows help slightly (window 4, 8, 16: 0.741, 0.748, 0.765). Wavelet cues do outperform Canny edge cues. All hand-built frequency features remain far below frozen DINOv2 features.

**RQ5. Does frequency-guided gated fusion beat concatenation?**

**Table 7. AUROC of fusion variants (DINOv2 plus frequency cues).**

| Model | Seen families (avg) | Unseen families (LOFO avg) | Unseen module_corruption |
|---|---|---|---|
| dino only | 0.962 | 0.837 ± 0.006 | 0.626 |
| frequency only | 0.874 | not run | not run |
| concatenation | 0.966 | 0.846 ± 0.002 | 0.632 |
| gated fusion | 0.966 | 0.851 ± 0.006 | 0.641 |

Gated fusion is nominally best and improves unseen-family AUROC by 0.014 over DINOv2 alone. However, its margin over concatenation (0.005) is within seed variation, and it is below DINOv2 alone on occlusion_warp (0.881 against 0.891). The evidence does not support a claim that gating outperforms simple concatenation.

**RQ3. How do cues degrade under noise?**

**Table 8. AUROC under noise (classifiers trained on clean images; 6,000 test images per condition).**

| Noise | sigma | fft_global | wav_multi | canny_topk | dino_ref | effnet_v2 |
|---|---|---|---|---|---|---|
| clean | 0 | 0.814 | 0.778 | 0.673 | 0.958 | 0.971 |
| Gaussian | 0.05 | 0.799 | 0.610 | 0.666 | 0.942 | 0.968 |
| Gaussian | 0.10 | 0.776 | 0.540 | 0.637 | 0.931 | 0.964 |
| Gaussian | 0.20 | 0.708 | 0.513 | 0.507 | 0.897 | 0.944 |
| Speckle | 0.05 | 0.808 | 0.624 | 0.670 | 0.945 | 0.970 |
| Speckle | 0.10 | 0.789 | 0.600 | 0.644 | 0.941 | 0.967 |
| Speckle | 0.20 | 0.759 | 0.528 | 0.548 | 0.922 | 0.957 |

Wavelet and Canny cues approach chance under noise, global FFT degrades gradually, and the learned EfficientNet is the most stable (0.971 to 0.944 at Gaussian sigma 0.20). Hand-built high-frequency cues are therefore the least noise-robust, which is the opposite of what a frequency prior is intended to provide. The noise scale (0 to 1 pixel range) is our choice and handcrafted features use a single seed.

**RQ4. How large are the distribution shifts?**
In DINOv2 feature space the MMD between plain and styled benign codes is 0.228, and between plain clean and plain camera images 0.310. The camera layer is thus a larger shift than benign styling. Pairwise heatmaps (MMD and sliced Wasserstein-1) over all class and condition groups are in notebook 5. These distances are descriptive and do not establish that shift causes transfer failure, which RQ6 tests directly.

**RQ6. Do features transfer across camera and style conditions?**

**Table 9. AUROC under controlled training-set ablations.**

| Features | Trained on | Camera test | Clean test | Plain benign vs tamper | Styled benign vs tamper |
|---|---|---|---|---|---|
| dino_ref | all | 0.940 | 0.972 | 0.957 | 0.962 |
| dino_ref | clean only | 0.561 | 0.971 | 0.759 | 0.778 |
| dino_ref | camera only | 0.940 | 0.948 | 0.942 | 0.947 |
| dino_ref | no styled benign | 0.689 | 0.669 | 0.982 | 0.475 |
| fft_global | all | 0.754 | 0.852 | 0.835 | 0.806 |
| fft_global | clean only | 0.626 | 0.858 | 0.765 | 0.701 |
| fft_global | no styled benign | 0.614 | 0.585 | 0.926 | 0.382 |

Training without camera effects leaves clean-image performance intact (0.971) but collapses camera-test AUROC from 0.940 to 0.561. Training without styled benign codes leaves plain-vs-tamper AUROC high (0.982) while styled-benign-vs-tamper falls below chance (0.475 for DINOv2, 0.382 for FFT): the model learns that styled means tampered. This is the controlled counterpart of the v1 diagnostic in Section 5.2 and supports the conclusion that coverage of benign variation in training is the dominant driver of generalisation in this benchmark.

## 6. Limitations

**External validity**
- All images are synthetic, and camera effects are simulated (mild in the main set, strong in the benchmark). No real phone photographs of printed or on-screen codes were collected. Real-world performance is therefore untested, and the generalisation gap is only partly closed. This is the main limitation.
- Images are 224 x 224. Very small module flips may partly vanish on downsizing, which may contribute to the weakness on module corruption.

**Baseline validity**
- The v1 baseline is a reproduction of the v1 data and recipe under the rules of this benchmark (9,968 training images), not the original 36,000-image model. Comparisons with v1 are comparisons with a reproduction.
- The unseen-family averages for v1 and aug cover three families, whereas the others cover five (Section 5.1 gives the matched three-family value for v2).
- The v2 gain is confounded: it combines benign hard negatives, the camera layer and more tamper families. Only the first two are isolated (Sections 5.2 and 5.4).

**Statistical validity**
- Reported standard deviations measure training randomness over three seeds. The split and the features are fixed, so they are not confidence intervals. For the frequency and DINOv2 methods the seed changes only the classifier.
- The DINOv2 pooling choice used validation LOFO AUROC, which involves validation images of all families (never test images).
- Handcrafted-feature results in the noise study use a single seed.
- Per-family metrics use benign images subsampled to the family size, so accuracy equals balanced accuracy on that subset.

**Scope of claims**
- Adversarial results are digital white-box attacks on undefended models, not physical attacks, and do not characterise a defended system.
- A clean substitute code, freshly generated with no visual artefact, cannot be detected by an image detector. Layer 1 does not address it and relies on the signature layer.
- Decode flags are stored in the manifest but are not analysed.
- The deep baselines use a single model family, because of limited compute (free or Pro Colab sessions) and independent working conditions.
- The companion study applies ideas from published work and does not reproduce it (Section 5.4).

## 7. Future work

1. **Real-camera evaluation set.** Collect 150 to 300 photographs of printed and on-screen codes with physical tampering, across devices, lighting and distances, and evaluate the existing models without retraining. This completes the third part of the v2 goal and quantifies the simulation gap. Collaborators with camera or print access are welcome.
2. **Controlled tamper-diversity baseline.** Train on the full benign data (styled, logo, camera) but only two tamper families, and test on the rest. Comparing this with v2 isolates the contribution of tamper-family diversity from that of benign coverage.
3. **Local detection of module corruption.** Evaluate higher input resolution and local crops or sliding windows, since downsizing may erase small flips and module corruption is the hardest family for every method.
4. **Calibration under family shift.** Study threshold selection and calibration for unseen families, since AUROC transfers better than the fixed operating point (F1 0.252 on module corruption).
5. **Adversarial hardening and physical attacks.** Evaluate adversarial training, and test printable perturbations, to determine whether the visual layer can be made robust at all.
6. **Full cascade evaluation.** Measure the combined detection and latency of the visual, signature and TLS layers, including the clean-substitute case that Layer 1 cannot detect.
7. **Decode-flag analysis.** Use the stored OpenCV decode flags to relate visual detectability to whether a tampered code still scans to the original or a different payload.
8. **Broader baselines.** Add fine-tuned vision transformers and fine-tuned DINOv2, and report confidence intervals from repeated group splits.

## 8. Reproducibility

Experiments ran on a Colab T4 GPU. The environment is recorded in `run_info.json`: Python 3.13, NumPy 2.4.6, pandas 2.2.3, scikit-learn 1.6.1, TensorFlow 2.21.0, image size 224, seeds 0, 1, 2.

```bash
git clone <GITHUB URL>
cd <repo>
pip install -r requirements.txt
sha256sum -c SHA256SUMS.txt        # verify downloaded dataset archives
```

Run the notebooks in the order given in Section 9. Each notebook is resumable, uses fixed seeds, and has a `SMOKE = True` switch for a quick check on a small slice with separate output folders. The generator is deterministic: the same seed and base ID reproduce the same images. Notebook 5 needs the features from notebook 1 and the saved models from notebook 3 (for the noise study).

## 9. Repository layout

> Adjust to the real archive before publishing.

```
README.md
LICENSE                           MIT (code)
CITATION.cff
requirements.txt
notebooks/
  1_feature_extraction.ipynb
  2_frequency_and_dino_lofo.ipynb
  3_efficientnet_baselines_lofo.ipynb
  4_adversarial_and_final_tables.ipynb
  5_frequency_priors_and_robustness_study.ipynb
generator/                        dataset generator and config.yaml
```

The dataset (manifest, 24 image shards, checksums and datasheet) is on Zenodo, not in this repository.

## 10. Ethics and intended use

The dataset is intended for research on visual QR tamper detection and its robustness. All payloads are harmless placeholders on reserved example domains and no real brand assets are included. It is not intended for producing attack material, and the adversarial experiments are reported to document weakness, not to supply a working attack.

## 11. Citation, license and acknowledgements

```bibtex
@dataset{<yourname>_qrshield_v2_2026,
  author    = {<Last>, <First>},
  title     = {QRShield v2: A Benchmark for Visual Tamper Detection in QR Codes Beyond the Training Distribution},
  year      = {2026},
  publisher = {Zenodo},
  version   = {2.0},
  doi       = {<ZENODO DOI>}
}
```

License: dataset and report CC BY 4.0, code MIT.

Acknowledgements: the companion study adapts methods from the HFP-SAM line of work (arXiv 2603.12708, Prof. Pingping Zhang's group), with SFMFusion (arXiv 2511.06593) and ISFM (arXiv 2602.04405) consulted at abstract and introduction level only. This was independent work without institutional compute.
