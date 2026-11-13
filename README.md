# QRShield v2

**A benchmark for visual tamper detection in QR codes beyond the training distribution.**

[Full report](docs/REPORT.md) | Dataset: <ZENODO DOI> | Code: MIT | Data and report: CC BY 4.0

QRShield v1 scored 100% on data like its training set, but only 84.9% on camera-degraded images and 51.7% accuracy on unseen tamper types. QRShield v2 is a synthetic benchmark and evaluation protocol built to measure and fix that generalisation gap.

- **66,083 images** from 12,000 base QR codes: five tamper families, styled and logo-bearing benign codes as hard negatives, and a camera-degradation layer.
- **Leakage-free protocol:** group split by base code, leave-one-family-out testing, thresholds set on validation only, three seeds.
- **Baselines:** fine-tuned EfficientNetB0 (v2), frozen DINOv2, global frequency features, and a v1 reproduction.

## Main result

AUROC, mean over three seeds:

| Test | v1 (repro.) | **v2** | DINOv2 (frozen) |
|---|---|---|---|
| In-distribution | 0.616 | **0.969** | 0.960 |
| Camera-degraded | 0.589 | **0.927** | 0.886 |
| Unseen tamper families | 0.554 | **0.854** | 0.840 |

The v1 failure came mainly from benign variation it had never seen (styled codes, camera effects), not from unseen tamper types. Module-level corruption remains the hardest family (0.661 AUROC when held out), and the model is not adversarially robust.

**Scope:** all images are synthetic and no real-camera test set is included, so these numbers describe the benchmark, not deployed performance. Full results, limitations and future work are in the [report](docs/REPORT.md).

## Use

```bash
git clone https://github.com/Memona-hafeez/QRShield_v2.git
cd QRShield_v2
pip install -r requirements.txt
```

Download the dataset from Zenodo (<ZENODO DOI>), verify with `sha256sum -c SHA256SUMS.txt`, then run the notebooks in `notebooks/` in numbered order (1 to 5). Each is resumable and seeded.

## Cite

See [CITATION.cff](CITATION.cff), or use the "Cite this repository" button on GitHub.

## License

Code: MIT. Dataset and report: CC BY 4.0.

Contact: Memona Hafeez, memonahafeez12@gmail.com, ORCID 0009-0007-4258-2386
