# Agent handoff: ForteBank action recognition and protocol monitoring

Updated: 2026-10-17 (Asia/Almaty). This is a conversation handoff, not a verbatim transcript. It records completed work, user intent, experimental findings, operational state and caveats. Verify mutable host/Git state before continuing.

## Current request and completion state

The user requested this complete Markdown summary for another agent. The immediately preceding work is complete: all three frozen encoders (VideoMAE v1, VideoMAEv2, CLIP) were rerun on the new RTX 5090, using the original train/validation split and train-only standardization. Reports were retrieved locally. No benchmark is currently running to our knowledge. No further training, relabeling, test evaluation or Git push is pending authorization from this request.

The user generally communicates in Russian and prefers action over repeated confirmation. They want practical visual inspection and reproducible evidence, not only architectural proposals.

## Objective and evolution of the conversation

The project concerns bank cashier videos and manual annotations of actions/possible violations under Kazakhstan banking procedures. The user initially asked to inspect the repository, PDFs, raw videos, Excel labels and manual review, then build a visualization to inspect clips/timestamps and common patterns.

Their core question is how to handle unfamiliar action sequences without equating unusual order with a violation. A customer may put money on the table, then provide a document, and the cashier may only take the money after checking it. A fixed sequence alone may be too restrictive. The user proposed constructing a golden dataset from normal segments before a violation and comparable normal segments from paired videos. That was an idea, not an established rule that all pre-violation frames are reliable normal labels.

The work progressed through:

1. Dataset inspection, video review viewer, annotation timelines, duplicate discovery.
2. Local video embedding comparisons for X-20/X-21, X-26/X-27 and X-4/X-5.
3. CLIP video/text action matching and protocol memory on X-26/X-27.
4. Separate money and cashier-presence detection, with CLIP retained for actions.
5. Frozen VideoMAE v1 versus CLIP action classifiers, with whole-video splits.
6. Train-only feature standardization ablation.
7. Baseline PR for collaborators; review of suggested ActionFormer, VideoMAEv2 and MMAction2 directions.
8. Hand/desk crop ablation.
9. Full-frame VideoMAEv2 comparison.
10. Visual diagnosis of hand-count versus machine confusion.
11. Preparation of a second GPU machine with RTX 5090.
12. Full three-encoder benchmark rerun on RTX 5090, validation only.

The conversation did not establish legal conclusions about ambiguous action order. Do not represent current action recognition as a validated legal violation detector.

## Workspace and instructions

- Local repository: `/Users/zhantore/Documents/projects/fortethon`.
- Shell: zsh on macOS. Use `python3`; `python` was unavailable in the user's shell.
- User instruction: **before every execution, read root `.agent/rules` and `.agent/workflows` files**. Both were absent during this work. Check again rather than assuming they remain absent. A prefix used was:

```sh
for p in .agent/rules .agent/workflows; do
  if [ -d "$p" ]; then find "$p" -type f -print -exec cat {} \;
  elif [ -f "$p" ]; then cat "$p"
  fi
done
```

- Preserve user changes and existing baseline artifacts. Do not reset/clean the working tree.
- Prefer small changes and existing inference utilities. Ponytail skill was active: `/Users/zhantore/.codex/plugins/cache/ponytail/ponytail/5.3.2/skills/ponytail/SKILL.md`.
- Do not spawn agents unless explicitly requested or applicable instructions require them.
- SSH/network required sandbox escalation in this session; authorized operations were approved. No unresolved approval rejection.
- Do not copy SSH private keys, platform secrets or cloud credentials to artifacts/Git.
- Do not evaluate test in subsequent exploratory experiments without a new explicit user request.

## Repository and collaboration state

- Remote repository: `https://github.com/ab-dauletkhan/fortethon/` (private).
- Current branch, verified when writing this handoff: `baseline/action-recognition-protocol-memory`.
- Existing PR: https://github.com/ab-dauletkhan/fortethon/pull/3 . Previously created, pushed and attached to this chat.
- Baseline PR commit recorded earlier: `f495570`, based on `origin/main` at `486829e`. Recheck before relying on these as current remote heads.
- Reviewers requested successfully earlier: Mustafa `Mustique23`, Bainazar `bainazarseitek`, Dauletkhan `ab-dauletkhan`.
- User had explicitly authorized creating the branch/PR and adding those reviewers. Later crop/v2/diagnosis/5090 work has **not** been committed or pushed during these turns.
- Original PR includes baseline scripts/viewers/configuration, README, dataset/embedding/roadmap documentation, and generated baseline snapshots under `docs/baseline-results/2026-10-17/`. `.gitattributes` collapses generated JSON/CSV diffs.
- `.gitignore` excludes `assets/videos/`, `review/generated/`, `open.zip`, Python caches. Local/remote generated results are not automatically included in PRs.

Working tree at handoff creation (before adding this file):

```text
 M README.md
 M docs/embedding-pilot.md
 M review/actions.html
 M scripts/run_action_pilot.py
?? data/action_crop.json
?? data/action_diagnosis.json
?? data/action_videomaev2.json
?? data/gpu_models.json
?? docs/benchmark-5090.md
?? docs/gpu-5090-setup.md
?? review/diagnosis.html
?? review/diagnosis.js
?? scripts/build_action_diagnosis.py
?? scripts/check_gpu_videomaev2.py
?? scripts/provision_gpu_models.py
?? scripts/run_action_5090.py
?? scripts/run_action_crop.py
?? scripts/run_action_videomaev2.py
?? scripts/vendor/
```

## Dataset and inspection

- 42 clips, 40 unique; aggregate duration ~1444.16 s, unique duration ~1299.86 s.
- Clip-level labels: 22 violation, 20 clean. These are not the same as the six action classes.
- Exact duplicates: X-4 = X-47; X-5 = X-48.
- H.264 video, 464×848, no audio, variable presentation timestamps. **Do not infer timestamps as frame index / 30.**
- Manifest: `review/generated/manifest.json`, with `clips`, `pairs`, `summary`.
- IDs in metadata use ASCII `X`; source filenames often use Cyrillic `Х`.
- Source videos: `assets/videos/`; source archive `open.zip` is a duplicate distribution archive, not needed when videos are extracted.
- Excel labels: `assets/open_labels.xlsx`, `assets/forte_labels_Mustafa.xlsx`.
- Manual chronological review: `docs/annotations.md`, rows with start/end/kind/types/seen/rule.
- `ground_truth.json` normal-step lists are not necessarily chronological.
- Case PDF: `Кейс_ForteBank.pdf`; context/documentation also in `docs/dataset-review.md`, `docs/model-contract.md`, `docs/baseline-roadmap.md`, `docs/embedding-pilot.md`, `docs/research/`.
- Review generation produced 1,060 sampled review frames.
- Main commands: `python3 scripts/build_review.py`, `python3 scripts/serve_review.py`.
- User initially hit missing `openpyxl`, installed it, then hit port 8766 already in use. Existing server was running; don't launch a duplicate or kill it casually.
- Server supports MP4 Range requests, `--port`, and graceful occupied-port handling.

Useful local URLs (server must still be running):

- http://127.0.0.1:8766/ — source clip review.
- http://127.0.0.1:8766/embeddings.html — early paired embedding visualization.
- http://127.0.0.1:8766/detections.html — detector experiment.
- http://127.0.0.1:8766/actions.html — action experiments; `#normalization` for normalization ablation.
- http://127.0.0.1:8766/generated/action-crop/report.html
- http://127.0.0.1:8766/generated/action-videomaev2/report.html
- http://127.0.0.1:8766/diagnosis.html#comparison
- http://127.0.0.1:8766/generated/action-5090/report.html — most recent report.

## Early embedding, detection and memory work

- `scripts/run_embedding_pilot.py`: VideoSwin representations for paired X-20/21, X-26/27, X-4/5; 1,024-dimensional features, 32 frames over ~2.13 s.
- `scripts/run_protocol_pilot.py`: CLIP actions on X-26/27, 2 Hz observations with a trailing 2 s window, table crop and state/memory logic.
- `scripts/run_detector_pilot.py`: GroundingDINO money/person observations using wide/patch views, retaining CLIP action scores.
- These were exploratory prototypes. In the prior detector/memory run, the requirement for two confirmed cash observations was not met, so the memory condition did not activate. This does **not** prove memory is useless; its benefit was not established with those observations.
- Do not interpret low text similarity to a negated prompt as a reliable absence detector.

## Frozen action benchmark contract

Sources: `data/action_pilot.json`, `scripts/run_action_pilot.py`, `review/generated/action-pilot/`.

- Six actions, in order: `hand_count`, `machine`, `laptop`, `document`, `safe`, `leave`.
- 2 s windows, 16 sampled frames (8 Hz), 1 s stride, 0.125 s annotation-boundary margin, maximum 8 windows per annotated interval.
- Frames selected from actual source PTS using floor lookup.
- Full frame preserves aspect ratio with padding to 224×224, bicubic resize, pad color `0x7b7468`.
- Native model pixel normalization retained.
- All encoders frozen/eval; heads are balanced logistic regression, lbfgs, max_iter=3000, seed 20261017.
- C grid `[0.1, 1, 10, 100]`, selected by validation macro F1, lower C wins ties. No refit on validation.
- 768-dimensional L2 window features for v1/v2/CLIP.
- v1: mean final spatial-temporal tokens from `MCG-NJU/videomae-base`, no mask/decoder at inference.
- CLIP: ViT-L/14, mean of L2 image projections over the 16 frames, then L2 window vector. The classifier uses visual embeddings, not text-prompt matching.
- v2: official `forward_features`, mean final patch tokens followed by learned `fc_norm`, then L2. K710 classification head is loaded strictly but not used as action predictions.

Counts:

| Split | Windows | Class counts in the order above | Sampled videos | Intervals |
|---|---:|---|---:|---:|
| Train | 123 | 3, 46, 22, 4, 34, 14 | 22 | 35 |
| Validation | 37 | 5, 6, 13, 2, 9, 2 | 6 | 14 |
| Test | 69 | 18, 13, 7, 5, 14, 12 | 9 | 20 |

Validation videos: X-18, X-19, X-20, X-21, X-22, X-23.
Test sampled videos: X-4, X-5, X-26, X-27, X-28, X-29, X-34, X-35, X-37. X-47/X-48 assigned to test but excluded as duplicates.
All paired/scenario-connected videos and exact duplicates stay in the same split. See config for complete assignments, including videos without sampled windows.

**Historical caveat:** the original unstandardized action pilot did evaluate test (macro F1 v1 ~0.1869, CLIP ~0.3596, text ~0.1101). Some footage had already been inspected, so test was not pristine. Subsequent normalization/crop/v2/5090 experiments excluded test. Never claim test has never been touched in the project.

Original generated manifest belongs to an earlier annotation snapshot (recorded as based on `b4cb042`); later main-branch annotation updates were retained. The selected 229 action windows were checked to be unchanged by row/label/time. Keep frozen experiment manifests rather than silently rebuilding from current annotations.

## Normalization ablation

- `scripts/run_action_normalization.py`; outputs `review/generated/action-normalization/`.
- StandardScaler mean/population variance fitted to 123 train embeddings in float64; transform to the feature dtype, no extra L2 afterward.
- Validation cannot influence scaler parameters; synthetic perturbation and constant-dimension checks passed.
- Original source artifact hashes preserved.

| Validation macro F1 | v1 | CLIP |
|---|---:|---:|
| Before standardization | 0.2824 | 0.5682 |
| Train-only standardization | 0.4030501 | 0.6613952 |

Selected C: v1 100, CLIP 10. Accuracy 15/37 and 23/37 respectively. Train macro F1=1 for both. Normalization helped but did not solve generalization.

## Crop ablation

- Config `data/action_crop.json`; script `scripts/run_action_crop.py`; outputs `review/generated/action-crop/`.
- Fixed camera crop `[0,240,440,640]` in source pixels, selected with train frames, ~1.9× zoom.
- Same 160 train/validation windows, PTS/frame indices and labels; no test preparation/evaluation.
- Separate train-only scaler per encoder/view, same C grid.
- v1 crop: macro F1 **0.45131965**, accuracy 17/37, C=100.
- CLIP crop: macro F1 **0.49407707**, accuracy 22/37, C=0.1. At fixed full-frame C=10, macro F1 ~0.39283.
- Both train F1=1, both hand-count recall 0/5.
- Conclusion: modest v1 gain, clear CLIP macro-F1 loss; crop did not generally help. Full-frame context matters. Camera-specific crop may remove useful context.

## VideoMAEv2 experiment

- `data/action_videomaev2.json`, `scripts/run_action_videomaev2.py`, vendor source/license/readme under `scripts/vendor/videomaev2/`.
- Official distilled ViT-base K710 checkpoint, unlike v1's self-supervised K400 checkpoint. Cannot isolate architecture effects.
- Upstream source commit `29eab1e8a588d1b3ec0cdec7b03a86cca491b74b`.
- Unmodified vendor source SHA-256: `d132c97369f25207e380e78c59294fd00b93d2f6aa0fb5c4ded19eff26a5a3cb`.
- Source MIT; HF weights repository metadata Apache-2.0.
- `timm==1.0.22`; official ImageNet normalization `(0.485,0.456,0.406)/(0.229,0.224,0.225)`. Do not use the unrelated generic config's 0.5 values.
- Register dynamically imported vendor module in `sys.modules` before executing it; required by timm registry. Registry overwrite/deprecation warnings observed and expected.
- Strict checkpoint loading, 710-class head shape, 86,773,190 parameters, no trainable encoder parameters.
- L4: 160 windows forward ~10.324 s, inference total ~17.134 s, model load ~3.87 s.
- Validation macro F1 **0.57472975**, accuracy 23/37, selected C=1 (tie with 10), train F1=1.
- C grid F1: 0.1→0.57239057, 1→0.57472975, 10→same, 100→0.48907292.
- Correct counts by class: `[0/5, 6/6, 5/13, 1/2, 9/9, 2/2]`.
- Machine precision is only 6/16 despite recall 6/6.
- Original output: `review/generated/action-videomaev2/`.
- Original feature file hash: `7935d5fd54f5118435745e86aa3b434517019824cfd687997478ab823a27603a`.
- Original prepared file hash: `146ddfad23a8853e069b5b452a96d2eb987350a7694d5eb16237a32468248c82`.

## Hand-count confusion diagnosis

- `data/action_diagnosis.json` contains provisional visual notes, not relabeling.
- `scripts/build_action_diagnosis.py`, `review/diagnosis.html`, `review/diagnosis.js`.
- `review/generated/action-diagnosis/`: 23 two-second review clips, montages, neighbor results, binary diagnostic parameters and preview.
- UI shows a validation query beside train hand/machine examples, raw cosine or train-standardized cosine, top five train neighbors, synchronized playback, speed and position controls. Synchronizing start times is not semantic phase alignment.
- Zoom is for visual review only; it does not change encoder inputs. Actual encoder thumbnails are available in details.

Three hand-count train examples:

| Index | Video | Window |
|---:|---|---|
| 3 | X-2 | 10.625–12.625 s |
| 5 | X-3 | 13.325–15.325 s |
| 18 | X-8 | 0.125–2.125 s |

They represent only **two related-scenario groups**; X-2/X-3 are related. Their raw cosine is ~0.946; similarities to X-8 ~0.570/0.441. Machine has 46 windows, 13 intervals, 12 videos, seven groups. All five validation hand windows come from one related X-20/X-21 group and overlap.

| Query index | Video/window | v2 prediction |
|---:|---|---|
| 96 | X-20 5.125–7.125 | document |
| 97 | X-20 6.125–8.125 | machine |
| 98 | X-20 7.125–9.125 | machine |
| 100 | X-21 8.125–10.125 | safe |
| 101 | X-21 9.125–11.125 | document |

Only two of the five errors are machine predictions. Queries 96/100 include sitting down, while 97 shows sustained hand work, so boundaries alone cannot explain all errors. Machine train example X-8 index 20 (7.125–9.125) looks largely idle while the machine runs. X-33 index 179 (7.125–9.125) largely shows chair movement/sitting. Broad episode labels can mix actions; these are review candidates, not automatically incorrect labels.

No query's top train neighbor was hand_count using either raw or standardized cosine. Raw best-hand/best-machine similarities were approximately .814/.849, .777/.828, .785/.857, .816/.838, .830/.833. Nearest-neighbor counts are biased by 46 machine versus 3 hand examples; cosine is also not the logistic decision function.

A separate binary diagnostic head used only 3 hand + 46 machine train windows, its own train-only scaler, fixed C=1 from the prior v2 experiment, no new validation search. All 11 validation windows (5 hand + 6 machine) were predicted machine: balanced accuracy .5, macro F1 .35294. Competition with document/safe is therefore not the only issue.

**Working hypothesis:** very sparse/diversity-poor hand training data plus broad/mixed action intervals are directly supported issues. The current embeddings + supervision transfer poorly. This experiment cannot causally separate representation limits from label noise/data scarcity. Class weights do not create new examples. Next useful experiment would refine action boundaries and add independent train scenes, keeping validation labels independent of predictions. This is a proposed next step, not work already authorized or performed.

## GPU machines and reproducible environment

### Original tmp-l4

```sh
ssh -o BatchMode=yes tmp-l4
```

- User `galymzhantore`; NVIDIA L4 ~23 GB.
- Project `/home/galymzhantore/fortethon-protocol-pilot-20261017`.
- PyTorch 2.7.1+cu128, transformers 5.19.0, numpy 2.2.6, sklearn 1.7.2.
- v2 timm 1.0.22 isolated under `.videomaev2-deps`.
- No remote FFmpeg in the earlier checks. Prepared arrays were transferred from local.

### New RTX 5090 host

User explicitly supplied and authorized this server:

```sh
ssh -i ~/.ssh/gcp_ed25519 -p 10367 root@79.116.80.166
cd /workspace/fortethon
source env.sh
```

Original supplied SSH command also had `-L 8080:localhost:8080`; no persistent tunnel or new service was established during setup.

- Vast.ai Docker container, not a kernel-level VM. Read `/etc/vast-agents-guide.md` before acting; this was read during provisioning.
- GPU RTX 5090, 32,607 MiB, compute capability 12.0; driver 610.57.04 at setup.
- Disk initially ~87 GB total, ~75 GB free after environment/model setup; recheck now.
- **No persistent workspace volume. Destroy/recycle loses files.** Stop/start was documented as preserving the container. Copy new experiment results locally before destruction.
- Project `/workspace/fortethon`; isolated venv `/workspace/fortethon/.venv`.
- Python 3.12.15, torch 2.7.1+cu128, torchvision 0.22.1+cu128; wheel includes `sm_120`.
- transformers 5.19.0, timm 1.0.22, numpy 2.2.6, sklearn 1.7.2, huggingface-hub 2.2.0, Pillow 12.3.0, matplotlib 3.11.2, openpyxl 3.1.5, opencv-python-headless 5.0.0.93.
- Existing `/venv/main`, system Python and host driver were not modified. `/usr/bin/ffmpeg` available.
- Full environment freeze: `review/generated/gpu-setup-5090/pip-freeze.txt` on host and locally.
- `env.sh` sets HF_HOME `/root/.cache/huggingface`, HF_HUB_CACHE `/root/.cache/huggingface/hub`, TORCH_HOME `/root/.cache/torch`, HF_HUB_OFFLINE=1, TRANSFORMERS_OFFLINE=1.
- Vast's original HF_HOME differs (`/workspace/.hf_home`), so activate `env.sh` before running model scripts.
- `/workspace/fortethon/env.sh` is host-specific and not a repository file.
- Do not inspect/copy `.vast_api_key` or expose services without the platform's authentication.

Provisioning documentation: `docs/gpu-5090-setup.md`.
Scripts: `scripts/provision_gpu_models.py`, `scripts/check_gpu_videomaev2.py`; manifest `data/gpu_models.json`.
Transferred 2,544 original project/data/artifact files (~1 GB), all SHA-256 verified. Included source videos, labels, prepared full/crop arrays, embeddings, fitted heads and viewers. Excluded Git metadata, duplicate `open.zip`, environments and secrets. Setup additions were copied separately. The transfer verification describes the initial snapshot; later README/docs/viewer edits intentionally change some files.

Weights were obtained as identical official copies, rather than copied from L4, and checked against the saved baseline hashes:

| Model | Repo/revision | File SHA-256 |
|---|---|---|
| VideoMAE v1 | MCG-NJU/videomae-base @ dc740ceda42fce44faed2ea03c6d447db72f6af9 | bc053ca2840a038b1068269a4eec06ca569689e9a1ed9376a5b2b8a111be5290 |
| CLIP | openai/clip-vit-large-patch14 @ 32bd64288804d66eefd0ccbe215aa642df71cc41 | a2bf730a0c7debf160f7a6b50b3aaf3703e7e88ac73de7a314903141db026dcb |
| GroundingDINO | IDEA-Research/grounding-dino-base @ 12bdfa3120f3e7ec7b434d90674b3396eccf88eb | 5548f844c928c4b6f411fa8cbcc2bfa8dbbba437cb1d513975519f93c2a9ed21 |
| VideoMAEv2 | OpenGVLab/VideoMAE2 @ 706cc172d65ebd4dedbee3f9c0183a93df9fa125 | 8141a6955e0700d11bf15928fe6d61e5cfe482606fed8cfdddb1b922c0fd88ec |
| VideoSwin | download.pytorch.org/models/swin3d_b_1k-24f7c7c6.pth | 24f7c7c6fd97dd52439d72c2208e0785081811389185a4781405b482404e4b9b |

v2 filename: `distill/vit_b_k710_dl_from_giant.pth`; first three use `model.safetensors`. The manifest is authoritative. Reprovisioning needs online access (unset offline flags); inference is offline.

Setup verification passed: pip check, CUDA matmul, torchvision CUDA NMS, strict frozen v2 inference on real train window index 3. Three float32 forwards produced finite 768D features, repeat consistency, cosine 1.0 to saved L4 feature, max absolute error 1.8440187e-7. Cold ~124.6 ms, warm ~19 ms, excluding decoding. Evidence: `review/generated/gpu-setup-5090/{models.json,transfer-check.json,smoke-test.json,pip-freeze.txt}`.

## Latest complete RTX 5090 benchmark

Script: `scripts/run_action_5090.py`. Documentation: `docs/benchmark-5090.md`. Output on host AND locally: `review/generated/action-5090/`.

```sh
cd /workspace/fortethon
source env.sh
python scripts/run_action_5090.py
```

**Do not blindly rerun:** script intentionally refuses if output directory exists. It is currently populated. A future rerun needs a separately versioned output location, not deletion of evidence.

Implementation reuses v1/CLIP `infer` and v2 `prepare/check/infer`, directing outputs to new subdirectories. Only the original 160 train/validation windows are included. Input links on the host are restricted to selected videos; local retrieval excluded `inputs/` symlinks because they contain remote absolute paths. Original prepared inputs remain locally in `action-pilot/inputs/`.

All encoders recomputed float32 embeddings on RTX 5090. Train-only scalers and balanced logistic heads fit on the host CPU. Saved parameters reproduce validation probabilities. Weight hashes, original window/input hashes, split checks, convergence, exclusion of test windows and baseline immutability checks passed. Downloaded prepared/features/pipeline hashes were rechecked locally.

| Action (validation count) | v1 F1 | v2 F1 | CLIP F1 |
|---|---:|---:|---:|
| Hand count (5) | 0.000 | 0.000 | 0.462 |
| Machine (6) | 0.667 | 0.545 | 0.667 |
| Laptop (13) | 0.000 | 0.556 | 0.455 |
| Document (2) | 0.222 | 0.400 | 0.444 |
| Safe (9) | 0.529 | 0.947 | 0.941 |
| Leave (2) | 1.000 | 1.000 | 1.000 |
| **Macro F1** | **0.4030501** | **0.5747298** | **0.6613952** |
| Accuracy | 15/37 | 23/37 | 23/37 |
| Selected C | 100 | 1 | 10 |

All three train macro F1 values are 1.0. **All 37 validation predictions of each model exactly match the earlier L4 results**; zero macro-F1 changes. GPU replacement did not change quality.

Forward totals over 160 windows: v1 3.288 s, CLIP 22.377 s, v2 3.096 s. Inference totals ~9.294, 26.553, 8.504 s respectively. These are not end-to-end/controlled throughput numbers: preprocessing/decoding excluded, timer boundaries differ slightly (v1/CLIP includes output CPU copy), warmup and loading matter.

Artifacts:

- `prepared.json`: frozen selected windows/config.
- `features.npz`: newly computed features for three encoders.
- `inference.json`: hardware/model/timing/provenance, nested v1/CLIP and v2 records.
- `pipelines.npz`: each train scaler and head parameters.
- `results.json`: train/validation metrics, per-class precision/recall/F1/support/confusions, C grid, scores and L4 comparison.
- `scores.csv`: 111 validation prediction rows (37 × 3).
- `validation.png`, `report.html`: visualization and per-action table.
- `v1-clip/`, `v2/`: intermediate per-encoder provenance/features.
- Remote log: `/workspace/fortethon/benchmark-5090.log`.

README and actions viewer link to this new report. They and benchmark documentation were synced to the new host after completion. No commit/push was done.

## Interpretation and next-agent cautions

1. CLIP + train-standardized linear probe is currently best on this validation set, not a universal winner.
2. Standardization improved both initial models. Cropping did not generally help. v2 improves on v1 but does not fix hand counting.
3. Strongest evidence points to inadequate independent hand-count examples and broad/mixed action annotations. Do not claim representation is conclusively at fault or conclusively adequate.
4. Repeated use of the same small validation set for C/model/experiment decisions limits generalization claims. Overlapping windows are not independent observations. No significance claims were established.
5. Perfect train F1 with much lower validation F1 signals poor transfer/overfitting in this setup, not a working violation detector.
6. Next proposed work: targeted action-boundary review, finer observable labels (preparing/alignment/counting/loading/unloading/waiting/sitting), and more independent train hand-count scenes. Preserve evaluation separation and avoid editing validation labels to fit predictions.
7. ActionFormer, VideoMAEv2 and MMAction2 were suggested by colleagues. VideoMAEv2 has been tried. Do not imply ActionFormer/MMAction2 were trained or validated here. Roadmap discussion is in project docs.
8. Protocol memory should distinguish observed state/preconditions from a rigid universal action order, and explicit absence from unobserved/uncertain evidence. No validated legal decision system has been delivered.
9. Preserve all old artifacts and local changes. Future PR updates should explicitly include new code/docs and selected lightweight result snapshots if authorized; generated videos/large arrays are ignored.
10. The current task is documentation only. Do not start the proposed new data/labeling experiments merely because they are described in this handoff.
