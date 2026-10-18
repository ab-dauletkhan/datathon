# Autoresearch protocol (action recognition, 6 classes)

Loop: **problem → hypothesis → candidate → compare to champion → noise check → adopt only if confident.**

## Hard rules
- Test split (X-4, X-5, X-26..29, X-34, X-35, X-37, X-47, X-48) is never loaded, encoded or viewed. Only the 160 train+val windows.
- Do not edit labels/annotations to fit predictions. Do not modify `evaluate.py`, `baseline_clip.py` or anything under `review/generated/action-*` (frozen evidence).
- Hyperparameter search only inside `train` passed to `fit_predict` (inner CV, grouped by `meta[i]["video"]` scenario). Never tune against evaluator output then re-run the same idea with a tweaked constant more than twice; that's validation overfitting — record it as such.
- Read `docs/agent-handoff.md` for context; check `.agent/rules` / `.agent/workflows` before executing.

## Evaluate
```sh
python3 scripts/autoresearch/evaluate.py scripts/autoresearch/candidates/<name>.py --note "<hypothesis>"
```
Appends to `docs/autoresearch/ledger.jsonl`. Verdict `win` = scenario-group bootstrap CI95 of CV ΔmacroF1 > 0 AND frozen-val Δ ≥ 0. `noise` = can't distinguish. `loss` = CI entirely < 0.
Champion: `scripts/autoresearch/champion.txt`. Only the orchestrator updates it, only on `win`.

Baseline (`baseline_clip.py`): CV macro F1 0.6014, val 0.6614.
**Champion since iteration 5: `siglip_late_clip.py` (CLIP + SigLIP late fusion): CV 0.7318, val 0.6773.** Champion hand_count CV F1 0.500 (baseline 0.105); hand_count = 8 windows, 5 videos, 3 scenario groups.

## Data available locally (CPU, fast)
`review/generated/action-5090/features.npz`: `clip`, `videomae`, `videomaev2`, each (160, 768) L2, window-mean.
`prepared.json` windows carry video, start/end, annotation bounds, source PTS, label.

## GPU (new features only)
- 5090: `ssh -i ~/.ssh/gcp_ed25519 -p 10367 root@79.116.80.166`, `cd /workspace/fortethon && source env.sh` (offline HF cache, videos in `assets/videos/`). No persistent volume — copy results back.
- L4: `ssh -o BatchMode=yes tmp-l4`, project `/home/galymzhantore/fortethon-protocol-pilot-20261017` (no ffmpeg; prepared arrays in `review/generated/action-pilot/inputs/`).
- New encoders must output the same 160 rows in `prepared.json` order to `review/generated/autoresearch/features/<name>.npz` locally (keys become `<name>:<key>`). Write remote outputs to a new dir under `review/generated/autoresearch/`; never overwrite existing output dirs. Don't expose services or copy secrets.

## Report (per candidate, append to `docs/autoresearch/log.md`)
Problem · Hypothesis · Change · CV F1 / Δ / CI95 · val Δ · verdict · one-line interpretation.

## Scale-up rules (added iteration 4, many agents in parallel)
- Feature files: write to the scratchpad first, then atomic `mv` into `review/generated/autoresearch/features/` (evaluators glob it concurrently). Only arrays with 160 rows there.
- GPUs are shared by several agents: use your own new output dir, keep batch sizes modest, check `nvidia-smi` before launching, never kill other processes.
- Many candidates are now being tested, so a single `win` can be a false positive. The orchestrator confirms any win before it becomes champion: (a) deterministic rerun, (b) neighbouring-constant stability (no isolated spike), (c) leave-one-video-out not worse than champion.
- Prefer one pre-registered candidate per hypothesis; at most 3.
- Stochastic candidates (random augmentation, MLP init, sampling) must average predicted probabilities over ≥5 fixed seeds inside `fit_predict`; a single seed can flip val Δ sign (seen in iteration 5: hand F1 0.09–0.40 from the seed alone).
