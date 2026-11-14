# QRShield v2

**A benchmark for visual tamper detection in QR codes beyond the training distribution**

[![DOI](https://zenodo.org/badge/DOI/10.5281/zenodo.24186683.svg)](https://doi.org/10.5281/zenodo.24186683)
[![Code: MIT](https://img.shields.io/badge/code-MIT-blue.svg)](LICENSE)
[![Data: CC BY 4.0](https://img.shields.io/badge/data-CC%20BY%204.0-lightgrey.svg)](https://creativecommons.org/licenses/by/4.0/)

[Report (PDF)](docs/QRShield_v2_report.pdf) | [Dataset and report on Zenodo](https://doi.org/10.5281/zenodo.24186683) | [Cite](#citation)

---

## Overview

QR codes can be tampered with after printing, for example by placing a sticker over a legitimate code, altering modules, or substituting a logo. QRShield is a layered defence against malicious QR codes, and this repository concerns its first layer, a visual tamper detector.

The first version (v1) scored 100% on data resembling its training set, but only 84.9% on camera-degraded images and 51.7% accuracy on tamper types it had never seen. QRShield v2 is a benchmark and evaluation protocol built to measure that generalisation gap and to close it.

**Contributions**

- A synthetic benchmark of **66,083 images** with five tamper families, styled and logo-bearing benign codes as hard negatives, and a camera-degradation layer.
- A **leakage-free evaluation protocol**: group split by base code, leave-one-family-out testing, validation-only thresholds, three seeds.
- A **one-factor diagnostic** that separates camera degradation, benign styling and unseen tamper families as causes of the v1 failure.
- **Baselines and robustness analyses**: fine-tuned and frozen-feature models, adversarial attacks, and a companion study of frequency-domain methods, including negative results.

## Results

AUROC, mean over three seeds (standard deviations are in the report).

| Test | v1 (repro.) | **v2 (EfficientNetB0)** | DINOv2 (frozen) | Global FFT |
|---|---|---|---|---|
| In-distribution test | 0.616 | **0.969** | 0.960 | 0.818 |
| Camera-degraded benchmark | 0.589 | **0.927** | 0.886 | 0.674 |
| Unseen tamper families (LOFO) | 0.554 | **0.854** | 0.840 | 0.766 |

Main findings:

1. **The v1 failure came mainly from unseen benign variation.** v1 labelled 98 to 100% of styled QR codes as tampered, and augmentation did not fix this. v2 keeps that rate under 7%.
2. **Training coverage matters more than architecture.** Removing styled benign codes from training lowers DINOv2 AUROC on styled-benign-versus-tampered codes from 0.962 to 0.475, below chance.
3. **Frozen features are competitive.** DINOv2 with a small MLP is within 0.014 AUROC of the fine-tuned EfficientNetB0.
4. **Module-level corruption is the hardest family** for every method (0.661 AUROC when held out).
5. **The detector is not adversarially robust.** PGD at 4/255 reduces accuracy from 0.937 to 0.006.

## Dataset

The dataset is archived on Zenodo ([10.5281/zenodo.24186683](https://doi.org/10.5281/zenodo.24186683)). It is generated from 12,000 base QR codes, with about five images each, and stored as 24 zip shards.

| Component | Images |
|---|---|
| Main set: train / validation / test | 41,958 / 5,810 / 12,232 |
| Camera-degraded benchmark (evaluation only) | 6,083 |
| **Total** | **66,083** |

Classes: `benign_plain`, `benign_styled`, `benign_styled_logo`, and five tamper families (`module_corruption`, `patch_overlay`, `logo_manipulation`, `sticker_overlay`, `occlusion_warp`). The camera layer (perspective, blur, glare, moire, JPEG, shadow, sensor noise) is applied to benign and tampered images alike so that degradation is not a shortcut cue. All payloads are harmless placeholder URLs on reserved example domains, and no real brand logos are used. See the [report](docs/QRShield_v2_report.pdf) for the full composition table.

## Evaluation protocol

- **Group split** by base QR ID (70/10/20), so that images from one base code never appear in two splits.
- **Leave-one-family-out (LOFO):** one tamper family is removed from training and validation and tested together with benign test images.
- **Thresholds** are chosen on the validation set only.
- **Metrics:** AUROC (primary), accuracy, balanced accuracy and F1, reported as mean and standard deviation over seeds 0, 1 and 2.

Please report results under the same protocol so that numbers are comparable.

## Repository structure

```
notebooks/
  1_Feature_Extraction.ipynb                  frequency and DINOv2 features
  2_Feature_Models_and_LOFO.ipynb             feature-based models, LOFO
  3_EfficientNet_Baselines_and_LOFO.ipynb     v1, aug and v2 baselines, LOFO
  4_Adversarial_and_Final_Tables.ipynb        FGSM and PGD, result tables, diagnostic
  5_Frequency_Priors_and_Robustness.ipynb     companion study (RQ1 to RQ6)
docs/
  QRShield_v2_report.pdf                      full technical report
CITATION.cff
LICENSE
requirements.txt
```

## Getting started

```bash
git clone https://github.com/Memona-hafeez/QRShield_v2.git
cd QRShield_v2
pip install -r requirements.txt
```

1. Download the dataset archives from [Zenodo](https://doi.org/10.5281/zenodo.24186683).
2. Verify them with `sha256sum -c SHA256SUMS.txt`.
3. Run the notebooks in numbered order. Each is resumable, uses fixed seeds, and has a `SMOKE = True` switch for a quick pipeline check on a small slice.

The notebooks were run on a Colab T4 GPU (Python 3.13, TensorFlow 2.21.0). Notebook 5 needs the features from notebook 1 and the saved models from notebook 3.

## Limitations

- All images are **synthetic**, and camera effects are simulated. There is no real-camera test set, so real-world performance is untested.
- The v1 baseline is a **reproduction** under the v1 data rules (9,968 training images), not the original model.
- The v2 gain is confounded between benign hard negatives, the camera layer and tamper-family diversity. Only the first two are isolated.
- Reported standard deviations reflect training randomness over three seeds. They are not confidence intervals.
- Adversarial results are digital white-box attacks on undefended models. A clean substitute QR code cannot be detected by an image detector, which is what the signature layer is for.

## Future work

1. A **real-camera evaluation set** of printed and on-screen codes with physical tampering.
2. A **controlled baseline** with full benign data but only two tamper families, to isolate the value of tamper diversity.
3. Higher-resolution and local detection for **module corruption**.
4. **Threshold calibration** for unseen families.
5. **Adversarial hardening** and physical attacks.
6. Evaluation of the **full cascade** with the signature and TLS layers.

Collaborators with camera or print access for item 1 are welcome.

## Citation

If you use this dataset or code, please cite it (see also [CITATION.cff](CITATION.cff)):

```bibtex
@dataset{hafeez_qrshield_v2_2026,
  author    = {Hafeez, Memona},
  title     = {QRShield v2: A Benchmark for Visual Tamper Detection in QR Codes Beyond the Training Distribution},
  year      = {2026},
  publisher = {Zenodo},
  version   = {2.0},
  doi       = {10.5281/zenodo.24186683}
}
```

## License

Code: MIT. Dataset and report: CC BY 4.0.

## Contact

Memona Hafeez | memonahafeez12@gmail.com | [ORCID 0009-0007-4258-2386](https://orcid.org/0009-0007-4258-2386)
