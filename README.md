# fortethon

AI service that checks cash-desk discipline in overhead bank-teller videos, plus the dataset tooling, annotations and regulatory research behind it. Jury document (Russian): [`docs/submission.md`](docs/submission.md). Server recovery: [`docs/runbook.md`](docs/runbook.md).

## Quick start

| what | where |
|---|---|
| web UI | http://79.116.80.166:10171/ (no login) |
| jury API | http://79.116.80.166:10171/api/v1 , header `X-API-Key: <API_KEY>` |
| health | http://79.116.80.166:10171/healthz |

The API key is handed to the jury separately and is never committed.

```bash
curl -X POST http://79.116.80.166:10171/api/v1/analyze -H "X-API-Key: <API_KEY>" -F "file=@video_017.mov"
# {"job_id": "3f2a9c1e...", "status": "processing"}
curl http://79.116.80.166:10171/api/v1/jobs/<job_id> -H "X-API-Key: <API_KEY>"
```

Poll until `status` is `done`. Every job finishes within 285 s of upload. If analysis did not finish, the job is still `done`, with verdict `violation`, one «Требуется ручная проверка» item and the reason in `error`. Accepted: `.mov` / `.mp4` up to 2048 MB. Full contract: [`docs/api.md`](docs/api.md).

### Run locally

Needs an NVIDIA GPU with 32 GB (tested on RTX 5090), Python 3.12 and [uv](https://docs.astral.sh/uv/). Two processes: the vLLM server and the backend.

```bash
# 1. VLM server, in its own venv (vllm==0.31.0, CUDA >= 12.8 wheels), weights in $HF_HOME
HF_HOME=/workspace/.hf_home HF_HUB_OFFLINE=1 vllm serve QuantTrio/Qwen3-VL-32B-Instruct-AWQ \
  --served-model-name vlm --host 127.0.0.1 --port 10301 \
  --max-model-len 26000 --gpu-memory-utilization 0.92 \
  --limit-mm-per-prompt '{"image":16,"video":1}' \
  --media-io-kwargs '{"video":{"num_frames":-1}}' \
  --mm-processor-cache-gb 0 --max-num-seqs 32 --enable-prefix-caching

# 2. backend
uv sync --no-dev
API_KEY=<API_KEY> DETECTOR=app.detector_vlm VLM_ACTIONS=1 \
  uv run uvicorn app.main:create_app --factory --host 0.0.0.0 --port 10100 --workers 1
```

`app/detector_vlm/deploy/vlm.sh` is the same vLLM command as the server runs it. Without a GPU, leave `DETECTOR` unset: the stub detector returns synthetic events derived from the file hash, which is enough to work on the API and the UI. Tests: `uv run pytest`.

### Environment

Backend (`app/config.py`):

| variable | default | meaning |
|---|---|---|
| `API_KEY` | empty | key for `/api/v1/*`. Required in production |
| `DETECTOR` | `stub` | detector module. Production: `app.detector_vlm` |
| `THRESHOLD` | `0.5` | min confidence for a catalog type to count as a violation |
| `THRESHOLD_OTHER` | `0.7` | same for type `other` |
| `JOB_DEADLINE_SEC` | `285` | final status deadline, counted from upload |
| `JOB_TIMEOUT_SEC` | `270` | max detector run per job |
| `POST_BUDGET_SEC` | `15` | time reserved after the detector for frames and JSON |
| `MIN_BUDGET_SEC` | `20` | a queued job with less detector time left fails as `deadline` |
| `APP_UPLOADS_ENABLED` | `true` | web UI uploads on/off. The jury API is unaffected |
| `APP_QUEUE_CAP` | `3` | max waiting web UI jobs |
| `MAX_UPLOAD_MB` | `2048` | upload size limit |
| `RETENTION_SEC` | `86400` | jobs and videos are deleted after this |
| `DATA_DIR` | `./var` | SQLite job store, uploads and results |
| `RULES_PATH` | `data/rules.json` | violation catalog with legal references |

Detector (`app/detector_vlm`):

| variable | default | meaning |
|---|---|---|
| `VLM_URL` | `http://127.0.0.1:10301/v1/chat/completions` | vLLM endpoint |
| `VLM_ACTIONS` | `1` | run the 3 s clip ACTIONS pass next to the per-frame STATE pass |
| `VLM_MIN_CONF` | `0.5` | events below this are dropped by the detector |
| `VLM_WAIT` | `900` | seconds to wait for vLLM at startup |
| `VLM_STATE_FRAMES` | `100` | max frames for the STATE pass |

vLLM (`vlm.sh`): `VLM_MODEL`, `VLM_PORT` (`10301`), `VLM_MAX_LEN` (`26000`), `VLM_GPU_UTIL` (`0.92`), `VLM_MAX_SEQS` (`32`).

## Research and dataset tooling

The sections below cover the dataset review tools and earlier experiments. They are not needed to run the service.

## Review interface

The web UI lives in [`frontend/`](frontend/README.md). Its built files are committed in `app/static/` and served by the backend at `/`, so running the API is enough. Rebuild with `cd frontend && npm ci && npm run build` after changing the frontend.

Design context: [`docs/product/ui-ux-plan.md`](docs/product/ui-ux-plan.md).

Start with the [baseline idea and roadmap](docs/baseline-roadmap.md), [recorded results without GPU setup](docs/baseline-results/2026-10-17/README.md), and [experiment log](docs/embedding-pilot.md). Obtain `open.zip` separately from the dataset owner and place it at the repository root; videos and model weights are not included in Git. The recorded baseline uses a frozen annotation snapshot; see its README before attempting exact reproduction with newer annotations.

The completed action run used Python 3.10, NumPy 2.2.6 and scikit-learn 1.7.2; GPU extraction used PyTorch 2.7.1+cu128 and Transformers 5.19.0. Local normalization used the same NumPy/scikit-learn versions. These are recorded environments, not a portable dependency lock. The scripts use Unix file locks and target macOS/Linux.

## Review the clips

The local viewer supports sequential playback, timestamped filmstrips, clickable manual observations, 20 comparison pairs, and an overview of all event timelines. It does not run a detector. Original spreadsheets and annotations remain unchanged.

```sh
# Requires Python with openpyxl and Pillow, plus ffmpeg and ffprobe on PATH.
python3 -m pip install openpyxl Pillow
python3 scripts/build_review.py
python3 scripts/serve_review.py
```

Open **http://127.0.0.1:8766**. The server listens only on this computer. Stop it with Ctrl+C.

If port 8766 is already in use and the viewer opens, the server is already running; there is no need to launch it again. To use a different port, run `python3 scripts/serve_review.py --port 8767` and open **http://127.0.0.1:8767**. Use `python3` and `python3 -m pip` together so packages are installed for the same Python interpreter.

The builder extracts `open.zip` into `assets/videos/` and creates ignored assets in `review/generated/`. Video previews are remuxed into MP4 without re-encoding. Frame timestamps come from decoded presentation timestamps, not frame number divided by 30. `review/index.html` can also be opened directly in a browser after building.

Read [the dataset inspection and ML framing](docs/dataset-review.md) for the file map, clip-by-clip review guide, label issues, and suggested evaluation.

## Local video embedding pilot

The pilot compares X-20/X-21, X-26/X-27 and X-4/X-5 using frozen Video Swin B features. It creates 2.13-second clips, records the actual sampled source timestamps, compares each query against its clean counterpart, and displays matched actions, nearest references, anomaly timelines, PCA and cosine similarity. It uses supplied manual labels; it is not a trained violation detector.

```sh
# First build the review above. Requires torch, torchvision, numpy, scikit-learn,
# matplotlib and Pillow, plus ffmpeg/ffprobe. No automatic package/model downloads.
# Official KINETICS400_V1 checkpoint must already exist at:
# ~/.cache/torch/hub/checkpoints/swin3d_b_1k-24f7c7c6.pth
python3 scripts/run_embedding_pilot.py
# Rebuild analysis/plots without running the model again:
python3 scripts/run_embedding_pilot.py --analyze-only
```

With the review server running, open **http://127.0.0.1:8766/embeddings.html**. MPS is used when available; otherwise CPU. Window selection and conservative label boundaries are in `data/embedding_pilot.json`. Generated clips, vectors, metadata and plots are in the ignored `review/generated/embedding-pilot/` directory. [Read the pilot results and limitations](docs/embedding-pilot.md).

## Text, table patch and causal memory: X-26 / X-27

Open **http://127.0.0.1:8766/protocol.html** for video-synchronized scores, a magnified table patch, action history, timelines and the comparison with/without memory. This experiment uses frozen CLIP ViT-L/14 image/text features, averaging past frame features for action descriptions. It is not a trained temporal video encoder or a general money detector. Both variants receive identical visual evidence; annotations are only read after predictions.

The completed run used the user's `tmp-l4` GPU host and its existing cached safetensors checkpoint. No inference API or new package installation was used. The two manually chosen regions only apply to this camera. Weights stay on that host; results and 95 sampled frames are in the ignored `review/generated/protocol-pilot/` directory.

```sh
# Rebuild local analysis from the saved features (no GPU or network needed):
python3 scripts/run_protocol_pilot.py analyze
# Check the causal memory logic:
python3 scripts/run_protocol_pilot.py check

# Full run on any machine with the cached checkpoint and dependencies:
python3 scripts/run_protocol_pilot.py prepare
python3 scripts/run_protocol_pilot.py infer
python3 scripts/run_protocol_pilot.py analyze
```

Preparation needs the existing review manifest, Pillow, numpy and ffmpeg/ffprobe. Inference needs PyTorch, Transformers and the complete cached snapshot `openai/clip-vit-large-patch14` at revision `32bd64288804d66eefd0ccbe215aa642df71cc41`; it runs strictly offline. Analysis needs matplotlib and scikit-learn. Prompts, regions and fixed heuristic thresholds are in `data/protocol_pilot.json`; the prepared snapshot is the authoritative configuration for each run.

To repeat on the already prepared GPU host, run `ssh tmp-l4 'cd ~/fortethon-protocol-pilot-20261017 && python3 scripts/run_protocol_pilot.py infer'`, copy `features.npz` and `inference.json` from that host's `review/generated/protocol-pilot/` back to the same local folder, then run local `analyze`. If configuration or input frames change, transfer the updated `scripts/run_protocol_pilot.py`, `prepared.json` and `frames/` first. Preserve the directory structure; the inference metadata verifies the prepared snapshot hash. A file lock prevents concurrent writes to an output folder.

The first run found no benefit from memory: both variants had AUC 0.979 but missed 6 of 12 violation time points at the fixed 0.5 threshold. This is one pair, not a held-out accuracy estimate. See the follow-up results in [the experiment notes](docs/embedding-pilot.md).

## Separate cash and cashier detection

Open **http://127.0.0.1:8766/detections.html** to compare Grounding DINO detections with the previous CLIP-only observations. The page includes original video, boxes and fixed zones, all low-scoring candidates, unchanged CLIP action scores, and timelines with/without memory. Both the initial full-table crop and the later exploratory close crop are preserved.

```sh
# Rebuild local analysis without running models:
python3 scripts/run_detector_pilot.py analyze
python3 scripts/run_detector_pilot.py analyze --table-view patch
python3 scripts/run_detector_pilot.py check

# Repeat inference on the already prepared own-GPU host:
ssh tmp-l4 'cd ~/fortethon-protocol-pilot-20261017 && python3 scripts/run_detector_pilot.py infer'
ssh tmp-l4 'cd ~/fortethon-protocol-pilot-20261017 && python3 scripts/run_detector_pilot.py infer --table-view patch'
```

Copy each host output `detections.json` back from `review/generated/detector-pilot/` and `review/generated/detector-pilot-patch/` into its corresponding local directory, then analyze. Inference uses the existing protocol pilot frames and `data/detector_pilot.json`. Analysis additionally reuses the old CLIP features/results without changing their actions or labels. Update the script/config on the host before rerunning changed code.

Required detector snapshot: official `IDEA-Research/grounding-dino-base`, revision `12bdfa3120f3e7ec7b434d90674b3396eccf88eb`, downloaded with `huggingface_hub.snapshot_download` selecting JSON, TXT and safetensors files. Once cached, inference is strictly offline using installed PyTorch, torchvision and Transformers; no remote model code or inference API. The initial runs used NVIDIA L4 in float32 and took about 35/33 seconds for 47 detector updates (two views each), excluding loading, transfer and local video decoding.

This is an exploratory two-video experiment, not a production detector. Full-table / close-crop results found 3/12 and 9/12 violation points, respectively, with no false alarms. Memory never activated because cash detections did not pass its two-confirmation condition; equal results do not establish that memory is unnecessary. Threshold calibration on independent recordings remains outstanding.

## Frozen VideoMAE vs CLIP: supervised actions

Open **http://127.0.0.1:8766/actions.html** for held-out action predictions, playable windows, the identical model inputs, confusion matrices, X-26/X-27 timelines and the exact video split. This experiment trains two small six-class logistic-regression heads, one on frozen self-supervised VideoMAE-base features and one on frozen CLIP ViT-L/14 features. CLIP text similarity is a separate zero-shot baseline. No encoder weights are updated.

The fixed configuration is `data/action_pilot.json`: six visible actions, explicit source annotation-row mappings, whole-video split, paired-scenario grouping, duplicate exclusion, prompts and C grid. There are 123 training windows from 22 videos, 37 validation windows from six videos, and 69 test windows from nine videos. X-26/X-27, X-4/X-5, X-28/X-29 and X-34/X-35/X-37 are entirely held out. Exact copies X-47/X-48 are assigned to test but never sampled. Every class occurs in every partition; some have very few independent examples.

```sh
# Local preparation: existing review manifest + ffmpeg/ffprobe, numpy, Pillow.
python3 scripts/run_action_pilot.py check
python3 scripts/run_action_pilot.py prepare

# Transfer to the same user-owned GPU workspace used for previous pilots.
rsync -az --relative scripts/run_action_pilot.py scripts/run_protocol_pilot.py data/action_pilot.json review/generated/action-pilot/prepared.json review/generated/action-pilot/inputs tmp-l4:fortethon-protocol-pilot-20261017/

# GPU features, then training and validation selection on that host.
ssh tmp-l4 'cd ~/fortethon-protocol-pilot-20261017 && python3 scripts/run_action_pilot.py infer && python3 scripts/run_action_pilot.py fit'

# Copy only results back; input frames already exist locally.
rsync -az --include='features.npz' --include='inference.json' --include='fit.json' --include='heads.npz' --exclude='*' tmp-l4:fortethon-protocol-pilot-20261017/review/generated/action-pilot/ review/generated/action-pilot/
python3 scripts/run_action_pilot.py report
```

For a completed run, only `report` is needed to rebuild the page data and plots. It does not refit the heads. `fit` uses scikit-learn; `infer` uses CUDA, PyTorch and Transformers with safetensors and offline-only loading. The completed run's model snapshots are cached on tmp-l4:

- `MCG-NJU/videomae-base@dc740ceda42fce44faed2ea03c6d447db72f6af9` (self-supervised Kinetics-400 pretraining, no Kinetics action head).
- `openai/clip-vit-large-patch14@32bd64288804d66eefd0ccbe215aa642df71cc41`.

On a fresh host, obtain those exact official snapshots with `huggingface_hub.snapshot_download`, selecting JSON/TXT/safetensors files, before offline inference. No inference API or remote model code is used. VideoMAE's [model page](https://huggingface.co/MCG-NJU/videomae-base) marks these weights CC-BY-NC-4.0; commercial deployment requires a separate licensing check.

Both models see the same 16 source frames per two-second window, with the full frame padded to 224×224 and model-specific normalization. Linear probes use L2-normalized 768-dimensional features, class balancing and the same C grid; C is chosen only on validation macro F1. There is no refit using validation or test. Hashes pin the input snapshot, weights, vectors and saved heads. `prepared.json` records actual source PTS and exclusions; `scores.csv` contains all predictions.

This is a closed-set action experiment on previously inspected footage, not a pristine external test, online violation detector or proof of OOD generalization. Unmapped/short/ambiguous intervals remain unscored. Coarse labels, small objects after resizing, correlated windows and sparse classes limit interpretation; see [the experiment notes](docs/embedding-pilot.md).

### Train-only normalization ablation (validation only)

Open **http://127.0.0.1:8766/actions.html#normalization** for the before/after comparison. This reuses the cached L2 embeddings and the unchanged original split. It fits `StandardScaler` mean and population variance on the 123 training windows only, then retrains the same balanced logistic heads. There is no second L2 normalization. All C values (0.1, 1, 10, 100), the selected validation scores and the scores at the original C are shown; no new test predictions are made.

```sh
# Local CPU; numpy, scikit-learn, Pillow and matplotlib already installed.
# The original action-pilot artifacts must exist. No video decoding or GPU run.
python3 scripts/run_action_normalization.py
# Rebuild only the plot, or run the scaler leakage/constant-feature checks:
python3 scripts/run_action_normalization.py report
python3 scripts/run_action_normalization.py check
```

Validation macro F1 on the same 37 windows / six videos: VideoMAE **0.2824 → 0.4031** (C 1 → 100), CLIP **0.5682 → 0.6614** (C 100 → 10; C 100 ties after scaling). With the original C held fixed, the normalized scores are 0.3372 and 0.6614. Validation accuracy changes from 10/37 to 15/37 for VideoMAE and 22/37 to 23/37 for CLIP. Both normalized heads fit train perfectly; the gap to validation remains. These are validation-selection results, not new test estimates.

Outputs are separate in `review/generated/action-normalization/`: `results.json`, `results.js`, validation-only `scores.csv`, `pipelines.npz` (selected head weights and per-feature train means/scales), and `validation.png`. Original artifacts are verified by hashes and left unchanged. Baseline predictions and every baseline validation C score are reproduced; saved pipelines reproduce probabilities; validation perturbation cannot affect fitted scaler statistics. The saved JSON contains only train/validation summaries and validation predictions, with `test_evaluated: false`.

### Fixed hands/workspace crop (validation only)

`data/action_crop.json` fixes one source-pixel rectangle `[0,240,440,640]`, chosen using train frames before inference. `scripts/run_action_crop.py` reuses the original prepared windows, source frame indices, PTS and split, and extracts only the 123 train / 37 validation windows. It never prepares or evaluates test windows. Each encoder/view gets its own train-fitted StandardScaler and the same balanced logistic head/C grid. The full-frame standardized baseline must reproduce exactly before comparison is accepted.

```sh
python3 scripts/run_action_crop.py prepare
rsync -az --relative scripts/run_action_pilot.py scripts/run_protocol_pilot.py review/generated/action-crop/prepared.json review/generated/action-crop/inputs tmp-l4:fortethon-protocol-pilot-20261017/
ssh tmp-l4 'cd ~/fortethon-protocol-pilot-20261017 && python3 scripts/run_action_pilot.py infer --out review/generated/action-crop'
rsync -az --include='features.npz' --include='inference.json' --exclude='*' tmp-l4:fortethon-protocol-pilot-20261017/review/generated/action-crop/ review/generated/action-crop/
python3 scripts/run_action_crop.py fit
python3 scripts/run_action_crop.py check
```

Open **http://127.0.0.1:8766/generated/action-crop/report.html** for metrics, class-level F1, train input examples and links to per-window validation predictions. `fit` runs locally on CPU; only frozen feature extraction needs the GPU. `report` rebuilds the static page/plot from saved results. Existing preparations are not overwritten: a changed crop requires a separately versioned experiment. Full-frame cached embeddings, original generated manifest and normalization results must exist and match their recorded hashes. The fixed crop is camera-specific and removes context useful for some actions.

### VideoMAEv2 base on the same full-frame inputs (validation only)

The v2 experiment uses the official **ViT-base distilled K710** checkpoint, not the self-supervised-only v1 pretraining recipe. This comparison evaluates complete pretrained representations; differences cannot be attributed to architecture alone. All three encoders have 768-dimensional features and use the same full-frame pixels, windows, split, train-only StandardScaler and logistic-regression C grid. Test inputs are not read by the new encoder.

The pinned model/checkpoint is in `data/action_videomaev2.json`; the inspected upstream model source and licence are in `scripts/vendor/videomaev2/`. On the GPU host, install `timm==1.0.22` into `.videomaev2-deps` (`python3 -m pip install --target .videomaev2-deps --no-deps timm==1.0.22`); existing PyTorch/torchvision dependencies are reused. Cache the exact official checkpoint using `huggingface_hub.hf_hub_download(repo_id='OpenGVLab/VideoMAE2', filename='distill/vit_b_k710_dl_from_giant.pth', revision='706cc172d65ebd4dedbee3f9c0183a93df9fa125')`. Loading uses `weights_only=True` and strict parameter matching. No inference API is used.

```sh
python3 scripts/run_action_videomaev2.py prepare
# Original action-pilot prepared.json and inputs must already be on this host.
rsync -az --relative scripts/run_action_videomaev2.py scripts/run_action_normalization.py scripts/run_action_pilot.py scripts/run_protocol_pilot.py scripts/vendor/videomaev2 data/action_videomaev2.json review/generated/action-videomaev2/prepared.json tmp-l4:fortethon-protocol-pilot-20261017/
ssh tmp-l4 'cd ~/fortethon-protocol-pilot-20261017 && PYTHONPATH=.videomaev2-deps HF_HUB_OFFLINE=1 python3 scripts/run_action_videomaev2.py infer'
rsync -az --include='features.npz' --include='inference.json' --exclude='*' tmp-l4:fortethon-protocol-pilot-20261017/review/generated/action-videomaev2/ review/generated/action-videomaev2/
python3 scripts/run_action_videomaev2.py fit
python3 scripts/run_action_videomaev2.py check
```

Open **http://127.0.0.1:8766/generated/action-videomaev2/report.html**. CPU fitting reproduces both old baselines from cached vectors before comparison; `report` rebuilds the page and plot from saved results. `prepare` refuses to overwrite an existing experiment. Original action-pilot artifacts and action-normalization results must exist locally; see their sections above. Inputs are the benchmark's 16 PTS-sampled frames over two seconds, not the model zoo's multi-clip/multi-crop evaluation recipe.

### Inspect VideoMAEv2 hand-count errors

With the previous v2 artifacts and source videos present, run `python3 scripts/build_action_diagnosis.py`, then open **http://127.0.0.1:8766/diagnosis.html**. This builds side-by-side inspection clips, all three train hand-count examples, nearest train neighbours in raw/standardized embedding spaces, original annotation text and explicitly provisional visual notes. A fixed-C binary train-only diagnostic checks whether confusion survives removing the other four classes; it does not change the original model or labels. Results are retrospective diagnostics on the existing validation set, not a newly selected baseline. See the final section of `docs/embedding-pilot.md` for evidence and limitations.

### RTX 5090: повтор полного action benchmark

[Результаты и воспроизведение](docs/benchmark-5090.md): все три encoder повторно запущены на GPU, прежние 123 train / 37 validation, scaler только по train. Test не обрабатывается. Запуск: `python scripts/run_action_5090.py`. Отчёт: http://127.0.0.1:8766/generated/action-5090/report.html .

### Autoresearch loop (validation only)

[`docs/autoresearch/SUMMARY.md`](docs/autoresearch/SUMMARY.md) summarises 51 pre-registered candidates scored with leave-one-scenario-group-out CV, the frozen validation split and a paired group-bootstrap noise check. [`log.md`](docs/autoresearch/log.md) records every hypothesis and result, [`ledger.jsonl`](docs/autoresearch/ledger.jsonl) every run and [`PROTOCOL.md`](docs/autoresearch/PROTOCOL.md) the rules. The one benchmark win (CLIP + SigLIP late fusion, CV macro F1 0.601 → 0.732) ties the CLIP baseline on 21 fresh windows, so it is unconfirmed. Test windows were never loaded.

```sh
python3 scripts/autoresearch/evaluate.py scripts/autoresearch/candidates/<name>.py --vs scripts/autoresearch/candidates/baseline_clip.py --no-log
```

The evaluator reads cached features from `review/generated/action-5090/` and `review/generated/autoresearch/features/`, which are git-ignored; regenerate them with `scripts/run_action_5090.py` and `scripts/autoresearch/extract_*.py` on a GPU host. Fresh-window results, feature provenance and the hand-crop contact sheet are committed in [`docs/autoresearch/results/`](docs/autoresearch/results/).
