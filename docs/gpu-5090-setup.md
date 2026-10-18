# RTX 5090 experiment environment

Prepared on 2026-10-17. Project: `/workspace/fortethon`; isolated Python environment: `/workspace/fortethon/.venv`.

```sh
ssh -i ~/.ssh/gcp_ed25519 -p 10367 root@79.116.80.166
cd /workspace/fortethon
source env.sh
python scripts/check_gpu_videomaev2.py
```

`env.sh` activates the project environment and sets `HF_HOME=/root/.cache/huggingface`, `HF_HUB_CACHE=/root/.cache/huggingface/hub`, `TORCH_HOME=/root/.cache/torch`, and offline Hugging Face/Transformers operation. It is specific to this machine and is not a repository file.

## Dependencies

Python 3.12; PyTorch 2.7.1 + CUDA 12.8 and torchvision 0.22.1 from the official CUDA 12.8 wheel index. CUDA 12.8 provides Blackwell support; the existing host driver was preserved. The torch version matches the L4 experiment. See [official PyTorch installation versions](https://pytorch.org/get-started/previous-versions/).

```sh
python -m pip install torch==2.7.1 torchvision==0.22.1 --index-url https://download.pytorch.org/whl/cu128
python -m pip install transformers==5.19.0 timm==1.0.22 numpy==2.2.6 scikit-learn==1.7.2 matplotlib Pillow openpyxl opencv-python-headless
python -m pip check
```

The exact resolved package versions are recorded in `review/generated/gpu-setup-5090/pip-freeze.txt`. FFmpeg is available from the system. System Python and `/venv/main` were not modified.

## Data and models

Copied `assets`, `data`, `docs`, `scripts`, `review`, README, ignore rules and the case PDF. This includes source videos and labels, prepared full-frame/crop inputs, frozen splits, embeddings, fitted classifiers and review pages. The duplicate `open.zip` archive, Git metadata, environments and credentials were excluded.

`data/gpu_models.json` pins official revisions and SHA-256 values for VideoMAE v1, CLIP ViT-L/14, GroundingDINO base, VideoMAEv2 base K710 distilled, and VideoSwin. `scripts/provision_gpu_models.py` obtains exact official copies and verifies them against the previous experiment hashes. It requires online access; unset `HF_HUB_OFFLINE` and `TRANSFORMERS_OFFLINE` before reprovisioning. Existing trained heads are included with the copied experiment artifacts.

## Verification

`check_gpu_videomaev2.py` runs offline, checks model/source/input hashes, strictly loads the weights, freezes the encoder, and runs three float32 GPU forwards on train window 3 (X-2, 10.625–12.625 s). It checks finite 768-dimensional features, repeat consistency, and agreement with the saved L4 feature. This does not retrain a classifier, change normalization or evaluate validation/test data.

Evidence is saved under `review/generated/gpu-setup-5090/`: `models.json`, `transfer-check.json`, `smoke-test.json`, and `pip-freeze.txt`. Generated files remain ignored by Git and are copied back locally.

This Vast instance uses container storage without a persistent workspace volume. Destroying/recycling it loses its files; save experiment outputs locally before doing so.

### Recorded GPU result

RTX 5090, compute capability 12.0; the installed wheel includes `sm_120`. The real-window test passed on `cuda:0`: cosine similarity to the L4 reference was 1.0, maximum absolute feature difference `1.8440187e-7`. Forward times were 124.6 ms cold, then 19.0 and 18.9 ms; these exclude decoding/preprocessing and are not an end-to-end throughput benchmark. CUDA torchvision NMS also passed. Existing whole-video/pair/duplicate split and train-only normalization self-checks passed.

All 2,544 transferred baseline/data files passed SHA-256 verification. All five model weight hashes matched. The provisioning and smoke-test scripts and this setup note were added separately after taking the baseline transfer manifest.
